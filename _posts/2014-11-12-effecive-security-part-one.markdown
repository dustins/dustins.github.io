---
layout: post
title:  "Effective Security: Part 1"
date:   2014-11-12 19:40:00
categories: code
published: true
tags: java spring shiro
---

Security is a crucial part to many projects you'll work on, especially when it comes to web sites and services. But implementing it, and doing it well, can be a daunting task when you go beyond URL based security towards securing access to individual objects. For most Java projects, you'll likely being using [Spring Security][spring-security] or [Apache Shiro][apache-shiro]. While I love [Spring Framework][spring-framework], I prefer Shiro's security model over Spring's implementation, and since it plays well with Spring Framework, it is my first choice. So for part one, I'm going run through providing security with Shiro. But don't fret if you are more interested in sticking to the Spring ecosystem, I'll be show that in part two.

Code for this post can be viewed and downloaded at [swigg-security-example][swigg-security-example].

## Benefits of Shiro

One of the things I like most about Shiro is that even in its documentation, it tries to persuade people from using the simpler ["implicit roles"][implicit-roles] that you'll regret if you're implementing security on a project that will continually evolve. In case you're lazy and don't click links, the basics are that it is an inevitability of simple security checks like…

{% highlight java %}
if (subject.hasRole("admin")) {
    // ...
}
{% endhighlight %}

…will quickly devolve into logical gymnastics as your boss promises a client features that doesn't fit with your original plan. In no time you'll end up with something like the following sprinkled throughout your code.

{% highlight java %}
if (subject.hasRole("admin") || subject.hasRole("editor")
    || (subject.hasRole("reporter-news") && !subject.hasRole("reporter-sports"))
    || article.getOwner().equals(subject.getPrincipal())) {
    // ...
}
{% endhighlight %}

Without a very simple, or impossibly well thought-out project, new roles or functionality will only cause this plague to spread throughout your code and become a nightmare to maintain.

I _&laquo;shudder&raquo;_ just thinking about it.

One of my biggest beefs with Spring Security is the slightly disparate way they want you to handle controlling access. Spring Security is full featured out of the box, but is more likely to cause you to write a hodgepodge of role checking for coarse grained authorization, and slowly roll in ACL checking for individual resources as your project grows.

Shiro not only discourages the false start of implicit roles, but provides an alternative path forward that can be used consistently for coarse and fine grained authorization.

## The Basics

If you want to understand the relatively simple concept of how permissions can be structured, you should definitely check out [Understanding Permissions in Apache Shiro][understanding-permissions-in-shiro], but here is a quick summary for the lazy.

[`Permission`][shiro-permission] is just an interface with one method [`implies(Permission)`][shiro-permission-implies], that checks if the provided permission is implied by the other. The most common implementation is defined as a string with three parts: a domain, a list of actions, and list of targets. The domain acts as the topic of the permission, the actions are a often verbs that can be performed, and the targets are the identity of individual instances that permission applies to. With a basic format of `<domain> : <action, …> : <target, …>` you end up with real implementations looking like this…

{% highlight text %}
article : read : *
article : update,delete : article-2
{% endhighlight %}

In the above, the first permission would represent the ability to perform a read (`read`) on any (`*`) domain object of type (`article`). Similarly the second permission implies the privileges to perform update or delete (`update,delete`) for a domain object of type (`article`) that has the identity of (`article-2`).

## Applying the Concept

There are three main ways to provide authorization checks with Shiro: through configuration files, annotations, or explicit permission checks. You're also able to use any combination of the above too. Here's an example of each respectively…

    # configuration in shiro.ini
    /article/** = perms["article:read:*"]

{% highlight java %}
@RequiresPermission({"article:read:*"})
public ModelAndView read(Article article) {
    // serve article
}
{% endhighlight %}

{% highlight java %}
public ModelAndView read(Article article, Subject subject) {
    if (subject.isPermitted("article:read:*")) {
        // serve article
    }
}
{% endhighlight %}

I personally don't like dealing with strings as permissions since you're more likely to have a typo or forget to update a check in the future. I like to have objects create permission strings for me, it makes mistakes far harder and refactoring far easier.

{% highlight java %}
public ModelAndView read(Article article, Subject subject) {
    if (subject.isPermitted(ArticlePermission.read())) {
        // serve article
    }
}
{% endhighlight %}

No matter how many roles get added to your project, unless the basic premise of what privilege you're looking for changes, you'll never need to change this code! It also is incredibly easy to write an implementation that checks permissions to read a specific article too.

{% highlight java %}
public ModelAndView read(Article article, Subject subject) {
    if (subject.isPermitted(ArticlePermission.read(article))) {
        // serve article
    }
}
{% endhighlight %}

## Working Example

If you're looking for a working example of this, definitely take a look at [SecurityTest][security-test] which is included in my [example project][swigg-security-example] for a more complete understanding of these classes, and to see how I do the plumbing with Shiro. But to keep this post short, I'm going to forego explaining that stuff. So lets get started with how we can combine the basic concepts into a security check.

The first thing we can do is create some roles…

{% highlight java %}
Role adminRole = new Role("admin")
Role memberRole = new Role("member");
Role guestRole = new Role("guest");
getRoleRepository().save(adminRole, memberRole, guestRole);
{% endhighlight %}

Followed by adding some users…

{% highlight java %}
User kermit = new User("kermit");
kermit.setRoles(adminRole, memberRole);
getUserRepository().save(kermit);

User fozzy = new User("fozzy");
fozzy.setRoles(memberRole);
getUserRepository().save(fozzy);
{% endhighlight %}

Now lets assign some permissions. I'm going to use the same permission implementation I use in my example project. You'll see it works much like the permission explained above, except that it has an additional parameter to represent the owner of the permission, and added overloaded methods for interfaces that allow the owner and targets to be given as objects instead of plain strings.

{% highlight java %}
// admins can perform any action, on any instance of any domain
Permission p1 = new DATPermission(adminRole, "*:*:*");
getPermissionRepository().save(p1);

// members can perform reads on all articles
Permission p2 = new DATPermission(memberRole, "article:read:*");
getPermissionRepository().save(p2);

// fozzy can perform reads and updates on all article-2
Permission p3 = new DATPermission(fozzy, "article:read,update:article-2");
getPermissionRepository().save(p3);
{% endhighlight %}

Now lets see how Kermit would fare compared to Fozzy…

{% highlight java %}
Article article1 = getArticleRepository().findById(1);
Article article2 = getArticleRepository().findById(2);

// as kermit
subject.isPermitted(ArticlePermission.read());   // true
subject.isPermitted(ArticlePermission.update()); // true
subject.isPermitted(ArticlePermission.update(article1)); // true
subject.isPermitted(ArticlePermission.update(article2)); // true

// as fozzy
subject.isPermitted(ArticlePermission.read());   // true
subject.isPermitted(ArticlePermission.update()); // false
subject.isPermitted(ArticlePermission.update(article1)); // false
subject.isPermitted(ArticlePermission.update(article2)); // true
{% endhighlight %}

You can see that Kermit is able to do everything, which all stems from the `*:*:*` permission he has since he is an admin. Fozzy however only has `article:read:*` and `article:read,update:article-2` so he can only update article-2.

## Conclusion

For a more complete example, you can take a look at [SecurityTest][security-test] and see a very similar example, but with more detail and all the code necessary in that repo to get this type of security working. Be sure to check back for an example of securing individual resources using Spring Security in the coming weeks!

If you have any questions, comments, or complaints, feel free to tweet to [@dpsweigart](https://twitter.com/dpsweigart)!



[spring-security]: http://projects.spring.io/spring-security/
[apache-shiro]: http://shiro.apache.org
[spring-framework]: http://projects.spring.io/spring-framework/
[swigg-security-example]: https://github.com/dustins/swigg-security-example
[implicit-roles]: http://shiro.apache.org/authorization.html#Authorization-Roles
[understanding-permissions-in-shiro]: http://shiro.apache.org/permissions.html
[shiro-permission]: https://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/Permission.html
[shiro-permission-implies]: https://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/Permission.html#implies(org.apache.shiro.authz.Permission)
[security-test]: https://github.com/dustins/swigg-security-example/blob/master/src/test/java/net/swigg/security/example/SecurityTest.java

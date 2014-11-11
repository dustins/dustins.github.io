---
layout: post
title:  "Effective Security: Part 1"
date:   2014-11-10 17:37:09
categories: code
published: true
tags: java spring shiro
---

Security is a crucial part to many projects you'll work on, especially when it comes to web sites and services. But implementing it, and doing it well, can be a daunting task when you go beyond URL based security towards securing access to individual objects. For most Java projects, you'll likely being using [Spring Security][spring-security] or [Apache Shiro][apache-shiro]. While I love [Spring Framework][spring-framework], I prefer Shiro security model over Spring's implementation, and since it plays well with Spring Framework, it is my first choice. So for part one, I'm going to cover providing security with Shiro. But don't fret if you are more interested in sticking to the Spring ecosystem, I'll be covering that in part two.

Code for this post can be viewed and downloaded at [swigg-security-example][swigg-security-example].

## Benefits of Shiro

One of the things I like most about Shiro is that even in its documentation, it tries to persuade people from using the simpler ["implicit roles"][implicit-roles] that you'll regret if you're implementing security on a project that will continually evolve. In case you're lazy and don't click links, basically it is an inevitability that checks like…

{% highlight java %}
if (subject.hasRole("admin")) {
    // ...
}
{% endhighlight %}

will quickly devolve into logical gymnastics as your boss promises a client features that doesn't fit with your original plan. In no time you'll end up with something like the following sprinkled throughout your code.

{% highlight java %}
if (subject.hasRole("admin") || subject.hasRole("editor")
    || (subject.hasRole("reporter-news") && !subject.hasRole("reporter-sports"))
    || article.getOwner().equals(subject.getPrincipal())) {
    // ...
}
{% endhighlight %}

Without a very simple, or impossibly well thought-out project, new roles or functionality will only cause this plague to spread throughout your code and become a nightmare to maintain. I _&laquo;shudder&raquo;_ just thinking about it.

One of my biggest beefs with Spring Security is the prominence of role checking in their documentation. Shiro not only discourages that dangerous implementation, but provides a nice alternative that is effective in both course and fine grained authorization checks.

## What We Can Do

Shiro doesn't do nearly as much as Spring Security out of the box, but it does provide the framework to make a wonderfully clean implementation. Both provide course grained authorization checks of URLs through basic settings and annotations that are implemented with [AOP][aspect-oriented-programming]. But what is more interesting to me is trying to secure individual resources. Instead of having to wrap complex conditionals around resources like I showed above, we can replace them with explicit and readable statements.

{% highlight java %}
if (subject.isPermitted(ArticlePermission.update(article))) {
    // ...
}
{% endhighlight %}

No matter how many roles get added to your project, the concept that you're checking if the current subject has an "update" permission for that specific article won't change!

The basic premise of the implementation is incredibly simple and can be thought of in three files—two interfaces ([PrincipalIdentity][principal-identity], [TargetIdentity][target-identity]), and [DomainPermissionEntity][domain-permission-entity] (which is a subclass of the Shiro provided [WildcardPermission][wildcard-permission]) so I can persist permissions to the database.

WildcardPermissions normally consist of three string parts, a domain would act as the type of thing you're dealing with, a list of actions that can be performed, and a list of targets the permission is valid for.

{% highlight text %}
<domain> : <action, …> : <target, …>
1. article  : read        : *
2. article  : read,update : article-2
{% endhighlight %}

The first permission might be added to the role "member" so anyone with the "member" role can read all articles. The second permission could be given to the user who created article-2. Assuming you have to be a member to contribute, we could remove the superfluous "read" action, but it doesn't hurt to have it.

A [DomainPermissionEntity][domain-permission-entity] is just a representation of that concept, with an added field denoting ownership. Targets come from calling getTargetIdentity() on classes implementing the interface [TargetIdentity][target-identity], while ownership which is similarly determined by calling getPrincipalIdentity() on classes implementing [PrincipalIdentity][principal-identity].

## The Basics

I welcome everyone to check out my [example project][swigg-security-example] to check out specifics of these classes and to see how I do the plumbing with Shiro, but to keep this post short, I'm going to forego explaining that stuff. So lets get started with how we can combine the basic concepts into a security check.

The first thing we can do is create some roles implement [PrincipalIdentity][principal-identity]…

{% highlight java %}
Role adminRole = getRoleRepository().save(new Role("admin"));
Role memberRole = getRoleRepository().save(new Role("member"));
Role guestRole = getRoleRepository().save(new Role("guest"));
{% endhighlight %}

Followed by adding some users which would also implement [PrincipalIdentity][principal-identity]…

{% highlight java %}
User kermit = new User("kermit");
kermit.setRoles(adminRole, memberRole);
getUserRepository().save(kermit);

User fozzy = new User("fozzy");
fozzy.setRoles(memberRole);
getUserRepository().save(fozzy);
{% endhighlight %}

Now lets assign some privledges…

{% highlight java %}
// admins can perform any action, on any instance of any domain
Permission p1 = new DomainPermissionEntity(adminRole, "*", "*", "*");
getPermissionRepository().save(p1);

// members can perform reads on all articles
Permission p2 = new DomainPermissionEntity(memberRole, "article", "read", "*");
getPermissionRepository().save(p2);

// fozzy can perform reads and updates on all article-2
Permission p3 = new DomainPermissionEntity(fozzy, "article", "read,update", "article-2");
getPermissionRepository().save(p3);
{% endhighlight %}

Now if we pretend we have two articles (which would implement [TargetIdentity][target-identity]), we can easily check if our users are authorized to perform different actions.

{% highlight java %}
Article a1 = getArticleRepository().findById(1);
Article a2 = getArticleRepository().findById(2);

// article : read : *
if (subject.isPermitted(ArticlePermission.read())) {
    // kermit could read all articles because he is an admin
}

// article : update : *
if (subject.isPermitted(ArticlePermission.update())) {
    // kermit could update all articles because he is an admin
}

// article : update : article-1
if (subject.isPermitted(ArticlePermission.update(a1))) {
    // kermit could update aritlce-1 because he is an admin
}

// article : update : article-2
if (subject.isPermitted(ArticlePermission.update(a2))) {
    // kermit could update article-1 because he is an admin
    // fozzy could because he has a permission targeting article-2
}
{% endhighlight %}

On the backend when isPermitted() is called, a list of target identities is checked that consists of the current logged in user, and all of their roles. Then the database is queried to see if any of those target identities own a permission that looks like the one being checked.

## Conclusion

For a more complete example, you can take a look at [SecurityTest][security-test] and see a very similar example, but with more detail and all the code necessary to get this type of security working. Be sure to check back for an example of securing individual resources using Spring Security!



[spring-security]: http://projects.spring.io/spring-security/
[apache-shiro]: http://shiro.apache.org
[spring-framework]: http://projects.spring.io/spring-framework/
[swigg-security-example]: https://github.com/dustins/swigg-security-example
[implicit-roles]: http://shiro.apache.org/authorization.html#Authorization-Roles
[aspect-oriented-programming]: http://en.wikipedia.org/wiki/Aspect-oriented_programming
[principal-identity]: https://github.com/dustins/swigg-security-example/blob/master/src/main/java/net/swigg/security/authorization/PrincipalIdentity.java
[target-identity]: https://github.com/dustins/swigg-security-example/blob/master/src/main/java/net/swigg/security/authorization/TargetIdentity.java
[wildcard-permission]: https://shiro.apache.org/static/1.2.3/apidocs/org/apache/shiro/authz/permission/WildcardPermission.html
[domain-permission-entity]: https://github.com/dustins/swigg-security-example/blob/master/src/main/java/net/swigg/security/authorization/DomainPermissionEntity.java
[security-test]: https://github.com/dustins/swigg-security-example/blob/master/src/test/java/net/swigg/security/example/SecurityTest.java
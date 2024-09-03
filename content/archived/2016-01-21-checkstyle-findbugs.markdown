---
layout: post
title: Checkstyle and findbugs
date: 2016-01-21
categories:
- code quality
---

Developers try to write code as good as possible. But we need some tools to validate. Checkstyle and findbugs are two great tools to check our code.

There are three ways to use Checkstyle in your development. Install plugin in your IDE maybe the easiest way to use. Although you can use the default config file to check the code, I highly recommend you to write a config file for you and your team. Because the default one has some noisy part which will affect your development. With default config file, it follows sun's rule strictly. You need to define a package-info.java in every packages, every variable needs a comment, all the method parameters should declared as final, etc. The comment part really bothers me. Actually I write pretty much comment in my code, but write so many comment makes me feel like, that, I am not writing code, more, I'm a documenter. 

And the constraint of method parameters is a little not reasonable. When I tried to switch two method parameters' value, I did want to change the assignment, and it's just for fun. Except for that, I can't think of any other situation witch I need to change the assignment of a method parameter. And add final to a method parameter just warns me do not assign new value to it. Which means it happens during the compile time. Unless for the inner class, there is no way we need to declare final in a method parameter. The final isn't the part of method signature.

And another one, I feel pretty uncomfortable is that Checkstyle checks inline conditionals. Are you kidding me...

Findbugs is really cool. It finds a performance problem where I use Integer.valueOf("xxx"). It turns out valueOf just call the parseInt method during the compile time. And one thing I was doing wrong for quite long time is I always use keySet to iterate a map. But when I need to access the key and the value in the same time, keySet way need to call get(key) method, and yes, get() method is O(1). But don't forget the conflict! If I use EntrySet to iterate a map when I use key and value in the same time, it will be better.



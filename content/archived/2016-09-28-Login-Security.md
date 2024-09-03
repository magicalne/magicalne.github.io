---
layout: post
title: Login Security
date: 2016-09-28
categories: 
- login
- security
- spring security
---
最近有机会深入了解一下神秘的登录了：）之前开发的规则引擎ui需要权限验证功能了，我就不得不写了一套权限验证的东西，从前端到后端。其实公司里是有集成的统一登录项目，但是这个项目做的不好，没有考虑好前后端分离场景下的权限验证，导致我走了很多弯路。   

先看看一般的登录场景   

1. 用户访问某一资源
2. 后台系统发现用户没有登录
3. 重定向到登录页面
4. 用户输入用户名和密码，并提交请求道后台
5. 后台验证用户名密码成功，跳转到登录成功后的页面

很多旧系统，特别是前后端没有分离的系统，基于用户名密码的验证方式，当用户验证成功之后，会将当前session标记为authenticated，这样当用户再访问后续资源时，只要带着cookie中的sessionId就可以了，而不需要后续验证。即，在session生效的时间段内，用户可以访问任何资源而不需要进行权限验证。这里称之为基于状态的验证（cookie）。   

另外一种，这里统称为基于token的验证，例如oauth2、basic authentication。每次访问的时候需要在header中加入token进行验证。   

以basic authentication为例。很长时间以来，我都认为basic auth是一种十分不可靠的权限验证方式，因为不像oauth2，token是从服务器临时获取而来的，basic auth的token是由用户名和密码经过base64加密而来的，众所周知base64是不安全的。但在我看过[这个视频](https://skillsmatter.com/skillscasts/5398-the-state-of-securing-restful-apis-with-spring)之后，我感觉我了解了真相。。。   

简单来说，这个视频是spring security的老大讲解rest api的安全问题。从这个分享，我认为Rob Winch是一个安全专家（至少在我的认知范围内是这样的）。深受启发呀。先抛出一个很有可能被忽视的事实：   
> the browser always sends cookies, and the server always has a session (unless you switch it off).    

回头再来看rest api的安全问题。从协议角度来说，rest api是无状态的，也就是说使用rest api访问资源不应该依赖session状态，而是依赖token。http是无状态的，但是在生产环境下使用http访问资源是十分危险的，最简单的例子是，当提交用户名密码时，在http下使用post请求，也是明文传输密码的。所以安全的传输必定是建立在https上的。而TLS/SSL是有状态的   
>The web server and the client (browser) cache the session including the cryptographic keys to improve performance and do not perform key exchange for every request.   
所以https下的rest api其实是stateful！所以正确的保护rest api的方式，是使用token＋session的方式。使用token保护rest api所访问的资源看起来十分自然，而使用session的原因在于csrf的保护，只要有浏览器访问rest api，就应该有防范csrf，因为浏览器每次发请求都会带上cookie，而且server总会维护session（除非主动关闭），所以使用基于session的方式放置csrf是很本能的做法。这里不需要再写其他的代码，只需要关注你的业务逻辑即可。   

这也就是为什么在我的后台spring服务中，想实现一个基于用户名和密码验证登录的rest api是如此的费劲。spring security中的核心就是filter，在各个filter中实现基于session、token的权限验证。用户可以自定义filter，或者扩展已有的filter。我发现，如果让rest api实现基于用户名密码的登录认证方式，还是可以使用HttpSecurity下的form－login的方式，但是需要重写sueccess handler和failure handler。这就使得rest api本身有了状态。当然这样没啥大问题，如果系统中的一些api需要暴露给外部系统调用的话，可以再为其设置basic authentication。多个authentication是可以同时存在的。   

针对spring security，这个框架使用起来要比spring mvc复杂的多，尽管这两个框架谁更复杂还不好说，但是作为程序员的使用情况是这样的。因为想authenticatino这种东西，基本上就是一次性配置，所以也没啥方便之说。比较坑的地方，我认为是spring boot-seurity starter中的configure dsl。这种   
```java
auth
	.xxxConfig()
	.and()
    .foo()
    .and()
    .end();
```
我感觉好不实用啊，还不如注入几个bean来的实在，而且官方文档也没有特别详实的描述，一路上也是踩了不少坑过来的。   

比如当我尝试使用basic auth的时候，我希望能加密用户名的密码，于是使用了推荐的bcrypt，我当时很天真的就把bcrypt的实现注入进来了，结果却没有生效，好气哦：（ 原因是如果使用默认的basic auth配置，验证用户名密码的部分是由spring security内部定义好的DaoAuthenticationProvider，而DaoAuthenticationProvider中使用的加密接口是一个废弃的借口，且不支持bcript，所以这一切你都必须在你自己的provider中实现。

---
layout: post
title: "cookie和csrf的故事"
date: 2016-10-20
categories:
- spring security
- csrf
- cookie
---

cookie是一个很古老的东西，但是直到今天我才发现，我还是很不了解cookie啊。   

### cookie path是干什么的？   

首先说明一点，cookie只认识地址，不认识端口号。而path指的是所在服务**页面地址**。例如，path=/foo，意味着，/foo和/foo下面所有的页面都可以读取cookie，但是path＝/bar或者path＝/都不能读取path=/foo下的cookie。   

注意这里说的是页面地址。现在大都是使用的restful接口，且都是前后端分离的，那么我向后台发送的/api/xxx所返回的set-cookie的内容，默认的path其实是后台的domain，假如后台的domain是api，则返回的path＝/api。注意如果在/home页面下的时候发送/api/xxx，其实cookie不会自动带上去的。   

### spring security中如何实现csrf保护   

首先，无论是否使用restful api，你的service都应该是有状态的，也就是基于token（cookie，jsessionId）的。默认的path值，就是service的domain。另外当登录执行成功的时候，response中会增加一个xsrf-token的cookie。spring security会对PATCH, POST, PUT, and/or DELETE这些修改状态的方法要求xsrf验证。验证方法就是在header中，（对，是自己在header中手动加入的）增加：x-xcrf-token:cookie.load("xcrf-token")。这个貌似是angularjs默认支持的header名字。

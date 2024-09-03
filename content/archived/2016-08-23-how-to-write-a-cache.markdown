---
layout: post
title: "How to write a Java cache"
date: 2016-08-23
categories: 
- cache
---

最近在看**Java concurrency in practice**   
真是一本好书哇。原本以为并发方面的书都会写的云里雾里的，没想到这本书是如此的大道至简。在书的第5章结束的地方，作者引入了一个问题：如何开发支持高并发的cache？我觉得很有意思，一直以来，cache在开发过程中时如此的不可缺少，一切的性能提升几乎都跟cache有关。   
那么如何开发一个属于自己的高xx缓存系统呢？   
书中例子使用了**ConcurrentHashMap**来保存cache，以此实现了高并发。另外所保存的value使用FutureTask<T>类型。这导致cache不能移除长久没有使用的对象，或者更新cache过的值。   
我在 [jache](github.com/magicalne/Jache) 这个项目中会一步一步的解决这写问题，最后写出一个真正的cache：）

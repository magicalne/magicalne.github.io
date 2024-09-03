---
layout: post
title: "Some dark side in Java -- Synchronize on Boolean"
date: 2016-07-30
categories: 
- concurrency
- synchronization
---
我们知道，java中的同步关键字是**synchronized**。它有两种用法：   

1. 修饰方法   
2. 修饰代码块   

修饰代码块需要给定一个在其上进行同步的对象，最理想的情况是对当前的对象同步：   
```java
synchronized(this) {...}
```
但往往我们并不是想要直接的给当前对象加锁，而是创造一个单一的、跟对象无关的临界区。
然后就有了这样的代码：   
```java
private Boolean aBoolean = ...
synchronized(aboolean) {...}
```
这个问题其实不是很明显, 原因是自动装箱和JVM对于常量的管理导致在一个jvm下，同时有且仅有两个Boolean实例：true/false   
尽管我可以
```java
new Boolean(true)
```
但是，我们知道，拜自动装箱所赐，这都是行不通的。   
这篇文章的作者有一个[例子](https://telliott.io/node/40 "sync1")   
```java
private Boolean isOn = false;
private String statusMessage = "I'm off";
public void doSomeStuffAndToggleTheThing(){

   // Do some stuff

   synchronized(isOn){
      if(!isOn){
         isOn = true;
         statusMessage = "I'm on";

         // Do everything else to turn the thing on

      } else {
         isOn = false;
         statusMessage = "I'm off";

         // Do everything else to turn the thing off

      }
   }
}
```
作者给出了解释
> Let's say we had two threads, T1 and T2. They both call doSomeStuffAndToggleTheThing at roughly the same time. T1 reaches the synchronize statement first and acquires a lock on isOn, which is currently false. T1 sets isOn=true and at this point, T2 reaches the synchronize statement, and can get a lock on isOn because it's now a completely different than the one on which T1 has a lock. So both threads are in the synchronized block at the same time, race to change our various attributes and leave the class in an inconsistent state.

简单说就是，两个线程同时访问这段代码，会因为同步对象改变而发生状态不一致的现象。   

另外一个类似的问题是，若有两段代码都对Boolean对象做同步，当多线程访问这两段代码时...   

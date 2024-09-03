---
title: GC是个什么鬼
layout: post
date: 2016-12-22
categories:
- java
- JVM
- GC
---

今天看了一个[视频](https://www.youtube.com/watch?v=we_enrM7TSY)。挺有意思。标题是：Understanding Java Garbage Collection and what you can do about it。Speaker是Gil Tene，他是Azul的Vice President of Technology and CTO, Co-Founder。

这个视频开头就埋了一个大大的伏笔。Gil问大家，各位所在公司开发的系统的服务器内存有多大？1/2 GB？ 1GB？ 2GB？ 4GB？ 10GB？ 20GB？ 50GB？？？细想起来这个问题真的好有意思啊。   

接下来就是各种扫盲。虽然关于GC也看了不少，但是学的不系统，知识总归没成体系。这个视频还是有一定点拨作用的。最重要的是，这个视频不是一板一眼的跟你讲GC，而是很诚实的说出来了GC没能做到的，或者说是很难做到的。然后你就可以以此来判断这个jvm实现是否够好了：）当然，那都是套路。。。   

视频说到，GC可能需要分代收集，不同代有不同的收集策略，而GC最终需要解决的就是**STOP-THE-WORLD**。如何尽量减少stop-the-world的频率和时间是JVM需要考虑的首要问题。然而这里面有各种各样的tradeoff。考虑到当年是2010年，当时的Hotspot的G1和CMS，并不是完全的concurrently，而是mostly。所以CMS还是会stop-the-world。   

这里又引出了另一个问题，理论上，如果可用内存足够大，永远不需要GC。而如果可用内存过小，就越需要GC。一个系统理论上，平均每秒会产生1/4G～1/2G对象，这些对象都是GC需要关心的。如果GC回收内存的速度比应用产生垃圾的速度快，那么应用自然会正常运行。一旦GC回收内存速度没能跟上应用产生垃圾的速度，应用就会慢慢死掉。如果应用所使用的server内存很大，stop-the-world所产生的影响会更加明显。所以就会碰到一个问题——Java application memory wall。就是说java 应用不能完全利用server的硬件资源，具体说就是越来越大的内存。   

那这个问题看起来好严重啊，内存越来越便宜，但是GC不够屌，只能用那么几G内存，太大了就性能不好。怎么办呢？   

1. JVM tuning
2. 完全不去stop-the-world，垃圾回收都是完全并行处理。

这个视频最后很有意思，峰回路转有提到了视频开头提到的问题，就是问各位开发的系统的服务器内存有多大？Gil所在的Azul开发的Zing使用的核心GC算法叫C4（Continuously Concurrent Compacting Collector），比当时的hotspot要屌的多。因为zing没有stop-the-world，是完全concurrently。然后对比jvm tuning，又把hotspot狠狠黑了一把。   

一般来说hotspot的jvm调优看起来是这样的：
>•Examples of actual command line GC tuning parameters:
•Java -Xmx12g -XX:MaxPermSize=64M -XX:PermSize=32M -XX:MaxNewSize=2g
• -XX:NewSize=1g -XX:SurvivorRatio=128 -XX:+UseParNewGC
• -XX:+UseConcMarkSweepGC -XX:MaxTenuringThreshold=0
• -XX:CMSInitiatingOccupancyFraction=60 -XX:+CMSParallelRemarkEnabled
• -XX:+UseCMSInitiatingOccupancyOnly -XX:ParallelGCThreads=12
• -XX:LargePageSizeInBytes=256m …
•Java –Xms8g –Xmx8g –Xmn2g -XX:PermSize=64M -XX:MaxPermSize=256M
•-XX:-OmitStackTraceInFastThrow -XX:SurvivorRatio=2 -XX:-UseAdaptiveSizePolicy
•-XX:+UseConcMarkSweepGC -XX:+CMSConcurrentMTEnabled
•-XX:+CMSParallelRemarkEnabled -XX:+CMSParallelSurvivorRemarkEnabled
•-XX:CMSMaxAbortablePrecleanTime=10000 -XX:+UseCMSInitiatingOccupancyOnly
•-XX:CMSInitiatingOccupancyFraction=63 -XX:+UseParNewGC –Xnoclassgc

然而对比Zing GC tuning，只需要这样:
>java -Xmx40g

理论上说，java程序员应该不需要理解GC调优，应该关注如何构建应用本身。但是事实上，这一代的程序员恐怕没机会享受这种待遇了。Zing的出发点很好，但是我相信一定付出了代价，不会有这么好的事，特别针对爱踩坑的程序员。总之这是一个可以颠覆认知的好视频，不怎么懂GC，就当好玩看看也行，作为GC入门也不错。但是不能太当真。

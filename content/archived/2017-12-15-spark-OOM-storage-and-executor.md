---
layout: post
title: Oh spark，贫穷限制了我的想象力
date: 2017-12-15
tags:
- spark
- OOM
---

两天前写好了一个一千行的spark sql，测试好了准备把结果写到s3上，结果写了两天没写进去，今天终于搞定了，特此记录。

### 事件回顾

这条sql不算复杂，左表10000+行，来回join两张千万级的表和两张不到一亿的表。中间所产生的内存开销为28.5G（从spark UI的input字段得出）。查询结果可以show出来，可以limit 100，但是limit 1000就不行了，更不能写到s3上。

报错信息是：

>spark.java.lang.OutOfMemoryError: Unable to acquire 44 bytes of memory, got 0
at org.apache.spark.memory.MemoryConsumer.allocatePage(MemoryConsumer.java:127)


我跟这个error纠结了两天。。。查了文档，jira，各种查，各种参数各种试，总是在最后一个stage弹出这个error。

spark版本是2.2.0，使用的是aws的emr，因为是28.5g的数据，我就开了core nodes是5台8 cores， 15g mem的cluster，这里的可用内存大概是55g，我心想怎么也该够用了吧。

今天试了2.0.0和2.1.1版本，都得出了相同的错误。还咨询了之前搞spark的同事，依然没搞定。

当我第三次阅读doc时，我隐约发现了问题所在。

### 什么是storage

这里就不介绍spark的内存管理了，贴个[简介](https://www.ibm.com/developerworks/cn/analytics/library/ba-cn-apache-spark-memory-management/index.html)。

简单理解storage，spark是把内存当硬盘用的，来持久化rdd，所以大多数情况，storage就是内存，但是当内存不够用的时候，就会出现spill to disk的情况，也就是写硬盘。所以如果storage足够大，内容就会被保存在内存中，反之就会spill to disk，当太小时，可能会出现频繁spill to disk，从而影响效率。

### 那么，executor的内存怎么办呢

executor会占用另外一部分内存，但是我现在不确定executor会不会spill to disk。至少凭借现在的现象观测下来，executor在shuffle write的时候是不会spill to disk的。这里也好理解，毕竟shuffle是要在executors之间转移的。

### 两个参数spark.memory.fraction和spark.memory.storageFraction

spark.memory.fraction默认是0.6

>Fraction of (heap space - 300MB) used for execution and storage. The lower this is, the more frequently spills and cached data eviction occur. The purpose of this config is to set aside memory for internal metadata, user data structures, and imprecise size estimation in the case of sparse, unusually large records. Leaving this at the default value is recommended. For more detail, including important information about correctly tuning JVM garbage collection when increasing this value, see this description.

spark.memory.storageFraction默认是0.5

>Amount of storage memory immune to eviction, expressed as a fraction of the size of the region set aside by spark.memory.fraction. The higher this is, the less working memory may be available to execution and tasks may spill to disk more often. Leaving this at the default value is recommended. For more detail, see this description.

也就是说，spark.memory.fraction让spark管理集群60%的内存，其中，storage和executor各占50%。剩下的内存大部分交由用户自定义数据结构，剩下一小部分系统保留。

spark 1.6之后引入了dynamic allocation，使得storage和executor之间可以动态调整，但是无论如何，默认情况下，spark只使用了60%的内存啊。

### 结论

由于spark只使用了60%的内存，而总的input是28.5g，总的可用内存是55g。55g * 0.6 = 33g，这确实足够跑出sql了。但是要是写文件到s3，是需要shuffle read各个rdd partition的，这时候，讲道理，我认为33g - 28.5g = 4.5g是足够读和写的。但是确爆出了OOM，所以我猜测spark这里实现方式是copy的。或者说，4.5g太理论值了，实际上的内存说会小的多？

总之，我用5台32g内存的机器就完美跑出结果了。。。真是像如今如火如荼的炼丹术（deep learning）一样，贫穷限制了我的想象力啊，难以想象google的算力！也就难以想象，我怎么就不知道用个大点的机器呢？？？就知道省钱。。。

在使用大机器跑的时候，我一直观察ganglia，发现，确实在进行到最后一个stage的时候（读partition并写文件），这里的内存使用量瞬间涨到了原来的两倍，这他妈不OOM才怪呢。

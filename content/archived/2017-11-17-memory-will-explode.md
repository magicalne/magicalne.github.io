---
layout: post
title: 妈呀，内存又爆了
date: 2017-11-17
tags:
- java
- G1GC
---

最近写了一个从postgresql到hbase导表job。pg中存的是text，有的条目还是很大的。

一开始没怎么注意，就用了200个线程导，发现很快内存就到上限了。回头看代码，也没啥问题，都是GC友好的。于是线程数降到100，内存还是慢慢的爆了。现象就是内存不断上涨，基本不下降，phoenix client在commit的时候会报奇怪的exception，这个时候job基本就不能工作了。挂上GC log继续跑了一次，发现GC基本无效，但还是有效的，只不过回收的很少。

回头再看业务，我是根据customer id出发导表的，每个id会有n条记录，每条记录是一个长度为m的数组，需要序列化整个数组，然后打平。我猜问题就出现在这里了，很多customer的记录会有上万条，每条记录的数组长度如果太长的话，这个序列化之后的list会很大。但是这些对象只是大而已，导的速度还是很快的，导完就可以扔掉了。

于是尝试使用G1GC，虽然没有都没有调大heap空间，但是效果明显改善，GC十分有效。

默认Parallel collector的log，连续几次的full gc：

```
6.783: [Full GC (Metadata GC Threshold) [PSYoungGen: 2880K->0K(424960K)] [ParOldGen: 8036K->8024K(52736K)] 10916K->8024K(477696K), [Metaspace: 20761K->20761K(1069056K)], 0.1111975 secs] [Times: user=0.11 sys=0.00, real=0.11 secs]
11.710: [GC (Metadata GC Threshold)
Desired survivor size 11010048 bytes, new threshold 1 (max 15)
[PSYoungGen: 413842K->7153K(424960K)] 421867K->22659K(477696K), 0.0436894 secs] [Times: user=0.04 sys=0.01, real=0.04 secs]
11.754: [Full GC (Metadata GC Threshold) [PSYoungGen: 7153K->0K(424960K)] [ParOldGen: 15506K->19928K(86016K)] 22659K->19928K(510976K), [Metaspace: 34780K->34780K(1081344K)], 0.2035734 secs] [Times: user=0.20 sys=0.00, real=0.20 secs]
15.594: [GC (Allocation Failure)
Desired survivor size 26738688 bytes, new threshold 1 (max 15)
[PSYoungGen: 417792K->10749K(555520K)] 437720K->67848K(641536K), 0.0464791 secs] [Times: user=0.03 sys=0.02, real=0.05 secs]
19.512: [GC (Allocation Failure)
Desired survivor size 68157440 bytes, new threshold 1 (max 15)
[PSYoungGen: 546821K->26089K(584192K)] 603919K->180072K(738304K), 0.3258985 secs] [Times: user=0.28 sys=0.04, real=0.33 secs]
19.838: [Full GC (Ergonomics) [PSYoungGen: 26089K->8672K(584192K)] [ParOldGen: 153982K->153842K(287744K)] 180072K->162515K(871936K), [Metaspace: 44692K->44684K(1089536K)], 1.2229773 secs] [Times: user=1.22 sys=0.00, real=1.22 secs]
...
360.402: [Full GC (Ergonomics) [PSYoungGen: 6450K->0K(619008K)] [ParOldGen: 437376K->292588K(472064K)] 443826K->292588K(1091072K), [Metaspace: 46611K->46611K(1091584K)], 1.8845454 secs] [Times: user=1.88 sys=0.00, real=1.89 secs]
```

fragment是gc的核心问题。尽管这里能看出，年轻代在grow、老年代在shrink。但是显然速度不够快，早晚会死掉。根据parall gc的文档：

> Growing and shrinking the size of a generation is done by increments that are a fixed percentage of the size of the generation so that a generation steps up or down toward its desired size. Growing and shrinking are done at different rates. **By default a generation grows in increments of 20% and shrinks in increments of 5%.** The percentage for growing is controlled by the command-line option -XX:YoungGenerationSizeIncrement=<Y> for the young generation and -XX:TenuredGenerationSizeIncrement=<T> for the tenured generation. The percentage by which a generation shrinks is adjusted by the command-line flag -XX:AdaptiveSizeDecrementScaleFactor=<D>. If the growth increment is X percent, then the decrement for shrinking is X/D percent.

也许改一下这个参数会有奇效？这里没有继续纠结，再看看G1的log。

首先是第一页log：

```
[Eden: 896.0M(896.0M)->0.0B(893.0M) Survivors: 8192.0K->11.0M Heap: 1222.7M(1508.0M)->329.3M(1508.0M)]
```

中间部分的log：

```
[Eden: 862.0M(862.0M)->0.0B(858.0M) Survivors: 12.0M->15.0M Heap: 1341.3M(1508.0M)->482.3M(1508.0M)]
```

以及最后的log：

```
  [Eden: 899.0M(899.0M)->0.0B(894.0M) Survivors: 5120.0K->10.0M Heap: 1137.9M(1508.0M)->243.4M(1508.0M)]
```

还是十分平稳的，这里没有用任何g1的参数，理论上调优空间还是很大的。



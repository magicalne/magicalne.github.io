---
title: 关于HBase
layout: post
date: 2018-06-04
tag: 
- HBase
- DB
---

用HBase也快一年了，记录一下心得和理解。

### Why HBase？

NoSQL之前还用过mongodb，当初选型就是HBase没跑了。关于各种NoSQL的特性讲解，可以参考那本很薄的《NoSQL精萃》。Mongodb是存document的，站在用户角度可以理解为JSON。很难想象Mongodb居然可以上市...这个东西很容易被用错。在上家公司供职的时候，有个同事说出了我对这个DB最大的concern：
>Mongodb写的时候不需要做额外操作，拿到Object就可以直接写。我们这个场景天然适合mongodb。

这里最大的问题就是，对于任何数据库，写不是最终目的，写是为了将来能够查询。如果不能方便查询，写是没有意义的。即使是写log，也要考虑将来是要以怎样的方式进行查询。

AWS上还有DynamoDB，看起来像是Mongodb，但貌似对标的是cassandra。

从业务考虑，主要存储目标是高表，可以考虑存储通话记录这种功能场景。一个人会有很多通话记录，我希望每个通话记录能作为一行进行存储。显然，HBase这种range scan的查询特性跟业务天然合拍。

### SQL or No SQL？

从当年google的big table，提出摒弃SQL，使用编程的方式替代SQL。再到big table退出历史舞台，google又搞了个spanner出来，重新请SQL走出来解决一切。可以看出，SQL，真的，很重要。而在HBase上写SQL，只有phoenix一家支持。

### Phoenix

Phoenix的代码质量很让人操心。构造方法的参数列表写了好几行的，且没有任何长度限制（120？不存在的）。核心功能在小版本出bug，而且还是AWS上的选用版本。我在phoenix上踩的坑都好憋屈。跑到jira上去看描述都能把我逗笑了。。。
但是没办法，还得用。

### CAP

HBase是CP，舍弃了A。C体现在CAS、行锁、多版本等一系列特性，P体现在region和region server。那么如果一台server挂了，怎么办？会丢数据吗？一台region server挂了，只会影响对这一台server的读/写。之前保存在挂了的这台region server的数据，只要硬盘不坏，数据就不丢。HBase的写是先写WAL再写memstore。

这里确实体现了HBase不支持HA的软肋了。当zookeeper与这台死掉的znode之间的session断掉一段时间后，zk认为这台server真的死掉了，会把这个znode删除。然后，会触发恢复操作。因为底层使用了HDFS，所以数据会有3份，可以用来恢复。下线的region发散到集群中的其他节点。但是由于待恢复的region的数据来源于集群中，数据的本地性很差，所有会有性能问题。这还是停留在理论上。我当初发现的问题是，一个server挂了，HBase根本就没有恢复过来。。。

### HA

那么就没有别的办法保证高可用了吗？勉强有一个，双集群，replica，也叫复制。详见jira：[10070](https://issues.apache.org/jira/browse/HBASE-10070)。估计是整个HBase中最复杂的部分了，能把读可用性提高的99.999%，当然写还是会挂。

### 其他

HBase的查询还是得靠缓存。所以业务上，一定要对cache友好。特别是HBase on AWS，HDFS后面还有一层EMRFS。没有命中缓存性能差了好多。

查询不要太复杂，3层left join顶天了。很多查询，我都是查询raw data，剩下的自己做处理。不要过于依赖phoenix提供的高级功能，能用基本功能替代就用基本功能吧，业务稍微妥协一下也是好的...

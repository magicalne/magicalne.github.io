---
title: 半年aws emr有感
layout: post
date: 2018-02-17
tag:
- AWS
- EMR
---

2017年6月末加入新公司，开始了一段startup之旅。核心技术自然是aws:)

我负责一个线上服务，后面拓展到了3个线上服务+一个hbase集群。多亏aws，我一个人就能hold住，几乎。

### EMR
因为主要工作内容跟data pipeline有关，所以EMR用的更多些。一开始并不是很理解EMR的设计，踩过几次坑之后，也就明白了。

EMR的核心思想是把存储和计算分离。

传统的技术架构，计算和存储耦合在一起其实也没什么问题。比如从主库抽数据到data warehouse，再在比如spark／hive上做计算。但是这里的问题就是，这个data warehuose必须一直online。当然对于大公司这没什么问题。但是对于一个小公司，只有一个人用，平常可能一星期才跑一次spark，如果一直online是很贵的。

所以EMR的玩法就是，所有的数据都放在S3或者Dynamodb上，当然AWS自己的其他基础设施也行。S3最适合我们的情况。计算就用spark或者hive。hive就用作导表，spark用于具体计算。hbase集群也是依托于S3的。

### EMR-HBase on S3
顺便一说，如果在生产环境下用hbase，一定要用on S3的模式，也就是存储是S3。而且要启动consistent view。这个consistent view用于集群和s3之间的一致性问题。怎么解决的呢？一致性用dynamodb再存一次咯。当然这里的dynamodb我们是访问不到的。

如果不用on s3模式，那就是一个玩具，随时会崩。虽然我无从得知具体的实现细节，但是可以猜测。hadoop那层还是要用hdfs的。而这层hdfs是在emrfs上面的。emrfs你是碰不到的，如果出现不一致的情况，只能让它崩溃了。这也就是consistent view存在的原因。

on S3的模式就不需要再像hdfs那样存3份了。hbase cluster会从s3 dir上读。实践下来还是有坑的。一致性的问题还是会有。不过稳定很多。

另外，hbase on s3就不再需要snapshot了，s3就是天然的snapshot。而且还可以加入replication cluster。但是这个replication是有损的。如果你用了phoenix on hbase，那么phoenix没法在replication上工作。我发现replication的实现方式是新增hbase:meta-cluster_id表。我记得hbase:meta这个表在master启动时会用到的，所以应该不是客户端配置的，通过修改指定hbase:meta表的方式，emr可以让你在只有一个s3 dir的情况下，为replication共享。用这种隔离系统表的方式，隔离replica。

### Data pipeline
我所有的data pipeline都是围绕hbase的。入口处还有几个postgresql上主站的表。

不得不吐槽，hbase和phoenix对spark的支持真是差到家了。。。导表还是hive好，可以直接把hbase的表导到s3。postgresql的表是用spark导的。最后用spark sql跑计算，并把结果写回S3。

spot instance是个好东西，可以省不少钱。

---
title: 乞丐版微服务之——AWS放大化
layout: post
date: 2019-12-03
tag:
- microservice
- aws

---

# 首先说几个问题。

1. microservice是一种架构模式；
2. 功能极致完善的软件系统，内部实现一定复杂且维护代价高；内部实现简单明了且维护成本低，那功能也会相对单一。这是工程师时刻需要trade-off的点；
3. 越来越多的公司上云了，aws、gcloud、azure、aliyun等等。每个云提供商面向的企业也不尽相同，往大了说有netflex，小了说有个人开发者。介于这里介绍乞丐版，那我们定义的目标客户就是小公司吧；
4. 大多数小公司的技术很渣，不足以看懂aws docs的level的渣；

# 小公司开发必备基础设施

既然都说是小公司了，如果不是纯技术创业，一般来说脱离不了互联网。而且不会有老板愿意掏钱让下面的人闲的没事了，开发个rpc框架，或是xx存储。

假如说要开发一个APP，那需要一个后端提供API访问。后台API除了需要做好load balance，还需要至少一个存储。这里少不了后台JOB，或者其他类似的生产者-消费者模式的服务。用户在APP内的行为数据也是要上报的，crash报告也是要上报的。

## 同步调用

AWS对于API后台提供了API gateway，后面可连接lambda，是一个较为完整的轻量级网关，不能做聚合，前置了cloudfront。优点是简单易用，最大的缺点我认为是贵。另外一种方案是自己做load balance，后台连多个ec2。麻烦了点，这个是最便宜的。

当初我开始接触AWS的时候，有一个问题百思不得其解，为什么AWS不提供rpc服务？AWS几乎所有服务都是基于HTTP+JSON，比如SQS、dynamodb等等。AWS提供通信的服务有API gateway、SQS、SNS。如果AWS要提供例如dubbo的RPC服务，那一定要有服务发现和服务注册。现在的EKS其实是有基于k8s+istio的配套服务的，但是以前是真没有阿。如今，我认为之所以AWS不提供RPC服务，主要原因就是因为AWS要给小公司服务，小公司用HTTP+JSON够了。再让小公司额外搞服务注册发现熔断。。。而且，lambda其实提供了同步调用的自动服务发现和服务注册。

首先，我们知道在开发过程中应该尽可能缩短调用链，以此来保证高可用性。也就是说要减少同步调用。这里的同步调用也没有必要使用API gateway，尽管支持private模式。这里完全可以用lambda替代。lambda很便宜，如果做一些没有什么上下文的事情，200行代码以内搞定所有事情，lambda是不二之选。我们可以在自己的服务中通过aws client调用lambda，这里注册发现就省下来了。那lambda出现异常要丢数据呢？熔断呢？这里也是可以做的，不过确实需要增加额外的开发量。可以为lambda中的error处理逻辑配置dead-letter-queue。这个dead-letter-queue意义就在于保证不丢数据。如果现有逻辑处理不了，报错了，扔到dead-letter-queue。后面再接cloudwatch alarm报警邮件，调好代码后重新消费dead-letter-queue中的消息。

dead-letter-queue不仅仅可以用于lambda，任何消费服务都需要配置。

服务间的同步调用，可以用lambda来实现，CRUD没问题，CQRS也可以。

## 异步调用

我们开发过程中经常会遇到这样一个问题，如果一个字段从状态A，变更为状态B，需要通知XXX。这种操作很常见，本身就是一个状态机。AWS提供了一个我认为很鸡肋的服务——step function。这个服务最好的一点是可以结合lambda。这样lambda不仅仅适合开发简单逻辑，也能适应复杂逻辑。但是单元测试是我非常非常注重的流程，而lambda肯定会降低UT覆盖率的。

那么我们还是把状态机放在自己的服务里吧。回到之前的问题，如何通知对方状态发生变化了呢？这种基于DB的服务，脱尾是不二之选。mysql读binlog，postgresql读wal，dynamodb可以stream。我们可以把最新的修改发到queue，让关心的服务去订阅，或者主动通知（lambda调用）。

核心技术就是SNS、SQS和kinesis。我们大部分服务使用SNS和SQS就够了，可以FIFO，也可以吞吐量优先，都是HTTP+JSON的接口。如果需要高级的stream处理，那就上kinesis。

## 存储

有了API gateway、lambda、sns、sqs，这个架构的形状就出来了。

说说存储。AWS提供了关系型RDS，主推二次开发的Aurora，可兼容mysql和pg，还有低版本serverless DB。serverless DB可以使用生产的snapshot快速拉起一个测试环境，跑完测试再杀掉，听上去还是很美好的。没有常驻实例也便宜不少，生产环境没听说有谁用。

Dynamodb，更像是cassandra的NOSQL，但是不是大宽表，最大支持400KB。适合存放用户信息数据，业务切入点与mongodb类似。读多写少，很便宜。

对于技术选型，Aurora和DDB都不需要考虑分库分表，如果事务第一，选Aurora，其他情况DDB真香。

NoSQL最大问题在于表设计，难度绝对大于关系型DB。如果业务场景多变，关系型数据库更像是以不变应万变的方案。而NOSQL需要想好方方面面的CRUD场景，罗列所有API，根据这些场景设计表结构。microservice架构的好，NOSQL才能香的起来。

我着实认为，大多数情况下，我们都不那么需要关系型DB，毕竟我们要开发微服务，NoSQL更友好。还有一个关键点，技术实例足够强，用啥都是用，技术都是工具。技术渣，再加上SQL，那就是面向SQL编程。一托SHI。

## 发布 + 监控

codepipeline可以从codecommit触发build和deploy。docker是必须的，逃不掉的。

k8s+istio不适合小公司，istio不适合大公司。。。

生产服务核心的问题是：滚动部署和auto scaling。ECS足够了，开几个ec2组成cluster，把自己的服务放上去。如果有必要，可以组一个spot cluster，便宜到炸。

cloudwatch新建metric，检索log中的关键字pattern，比如“Error”，“Exception”。然后设置Alarm并触发报警邮件。

如果想再完善，可以在特定的branch部署完后跑集成测试。

## ETL

不管是binlog、wal还是ddb stream，所有数据都可以通过kinesis firehose实时同步到S3，增量的全量数据。可以用spark还原原始表。

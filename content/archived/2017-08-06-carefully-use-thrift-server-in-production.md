layout: post
title: 谨慎在生产环境使用thrift server
date: 2017-08-06
tags:
- java
- thrift
---

在生产环境引入thrift server一周之后，被逼无奈，使用grpc+protobif替换掉了thrift。

使用thrift IDL的场景很多，特别是大厂们，不同语言的异构系统，可以通过thrift无缝衔接。但是进程间如何通信呢？RPC框架的选择真的少啊。我不想直接上dubbo，太重了，这只是一个简单的gateway。由于一开始调研的时候发现了一个有趣的事情，导致我一上来就放弃了grpc。protobuf很早之前就开源了，但是grpc是在近几年才开源的，然而grpc在google内部其实是和protobuf同年在内部release的。我对于google的半截子工程产生了深深的怀疑...然后就相信了thrift，也没有仔细看thrift的论文。

### thrift server的问题

其实也不能算问题吧。thrift server底层是c++实现的，设计之初就是应用于响应时间短的**快**请求。也就压根没有对**慢请求**有任何的优化，而且client与server的通信机制也十分简单，很多异常情况也没有考虑实现。

这就导致，你的请求不能慢，你必须转换成快请求。那thrift server是否实现了我们习以为常的**异步**接口呢？答案是：Yes, and no!

thrift client和server都有nonblocking的实现，但是这和我们基于java，特别是netty的nonblocking不太一样。关于异步实现的解释基本上很难在网上找到了，很多文章都在错误的演示async client的使用方式。当然你也不能在官方文档中找到，只能通过论文的设计的原理一探究竟...

当你实现了一个async client rpc方法，当你尝试使用Future包装并发送请求时，你会惊讶的发现，你只能以阻塞的方式的调用，而不能以非阻塞的方式调用，也就是不能异步的调用。

你需要:
```java
Object result = future.get();
```

当你尝试使用CompletableFuture包装，并apply到另一个线程处理的时候，你会惊讶的发现，请求根本发不出去的！你必须以阻塞的方式进行调用。

thrift server期待的nonblocking是使用oneway修饰的rpc方法。如果你的rpc不需要返回值，可以设置为oneway，并且使用async client发送oneway rpc请求。也就是说，只有onway修饰的方法才能使用async client，而oneway修饰的方法意味着没有返回值。thrift希望快请求再快一点。这也就是为什么async client发送异步请求时，结果跟我们期待的如此不同。

另外一个严重的问题是：thrift client没有pool。这是很严重的。网上有一些介绍，如何使用commons-pool实现的例子，但都太简单了，是不能在生产环境使用的。最严重的一个问题是，当server shutdown重启之后，pool中的连接应该已经断掉了，如果你没有实现检测远程连接是否断开的逻辑，pool中的连接是不能访问server的，你会得到一片片的异常。究其原因，是因为tcp头上的id对不上号导致的。当然还有没有其他问题，我就不得而知了。

这一波必须跪舔protobuf+grpc。特别是grpc，底层还是使用了netty，client会帮你维护心跳，有pool，简单且足够用。
---
title: 初探io_uring
layout: post
date: 2021-02-08
tag:
- io_uring
---

linux以性能著称，但是在一些小于1%的场景下，linux在io的表现上并不能让人满意。

主要原因在于上下文切换。

无论是网络请求还是本地磁盘io、read/write、pread/pwrite、epoll、aio都绕不开频繁的上下文切换。这是linux的`by design`。

io执行一次read操作，一定需要从user space转换到kernel space。为了给网络io增大吞吐量，通常会使用增加线程数量来处理大规模的并发请求。

但是一些要求低延时的场景，过去linux提供的这些io操作不能满足需求。业界解决方案就是[DPDK](https://www.dpdk.org/)。

网络操作因为linux的设计导致不能满足需求，那就绕过去。自己实现tcp、udp协议栈，使用专业网卡，甚至fpga直接烧。

对于这个问题，linux也给出了自己的解决方案：`io_uring`

io_uring从用户角度看，设计是异常简单明了的。io_uring将一次io操作进行拆分，拆分成提交请求和接受回复。

二者均使用了`ring buffer`的数据结构，以及`生产者-消费者`模式。

注意这里有两个ring buffer，提交和完成两个阶段使用不同的ring buffer。

提交请求就是，生产者（客户）向ring buffer发送一个消息，消费者（kernel）从ring buffer读取一个消息。
当kernel处理完之后，生产者（kernel）向完成队列提交一个消息，消费者（客户）从ring buffer读取一个消息。

这里性能提升的来源在于，由于ring buffer是一个共享的数据结构，user space和kernel space不需要不断的切换。

io_uring还提供了其他高级功能，比如异步通知。

因为linux的读写操作都是阻塞的。之前写了一个container，里面起了一个linux shell，我希望在host上执行container linux的shell操作。

我希望知道，ptmx是否有新内容可以读，以及是否我想提交命令到container linux。

这种功能epoll可以很好的完成，但是不高级，是时候祭出io_uring了。

[code](https://github.com/magicalne/libcontainer-rs/blob/172f53e60a6a110ab666d5d0474fe75cadf9e7f8/src/linux.rs#L632)

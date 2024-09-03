---
layout: post
title: Paper -- "The java.util.concurrent Synchronizer Framework"
date: 2016-12-10
categories:
- java
- concurrency
- AbstractQueuedSynchronizer
---

最近在看Doug Lea的那篇基于AQS实现的理论[论文](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)。陆陆续续看了几遍。论文的问题就是作者会把一个或者一类具体问题进行抽象和总结，针对这个或这类问题进行研究，并试图站在一个更高的位置得出结论。这就导致，读这篇论文还是有点难的。主要的难点是细节理论的理解。毕竟这些论文看的少，很多上下文不是很懂，但是若是为了彻底理解这些，把引用的论文都看一遍，有点累。。。   

我还是从作者的出发点开始理解吧，试图理解整个AQS为什么被设计成这样的。后续会跟着写几篇关于java memory model，reorder，barrier的文章，试图把整个多线程技术，从最顶层java实现到最底层现代cpu、寄存器、内存全部串起来。另类撸串：）   

### Introduction   

首先看看什么是synchronizer——同步器。所谓的synchronizer就是用来维护一组状态的变量，在java1.5发布的AQS中，就是一个32位的int，后面在java1.6发布的时候，又开发了基于long 64位的sycnronizer——AbstractQueuedLongSynchronizer。   

>Among these components are a set of synchronizers –
abstract data type (ADT) classes that maintain an internal
synchronization state (for example, representing whether a lock
is locked or unlocked), operations to update and inspect that
state, and at least one method that will cause a calling thread to
block if the state requires it, resuming when some other thread
changes the synchronization state to permit it. Examples include
various forms of mutual exclusion locks, read-write locks,
semaphores, barriers, futures, event indicators, and handoff
queues   

同步的处理涉及两种方法：acquire和release。具体所使用的同步方法可能不同，但都不会超出acquire和release的范畴。例如：   
>methods Lock.lock,
Semaphore.acquire, CountDownLatch.await, and
FutureTask.get all map to acquire operations in the
framework.   

### Design and implementation   

下面看一下关于AQS设计实现的伪代码：   
acquire:   
>while (synchronization state does not allow acquire) {
  enqueue current thread if not already queued;
  possibly block current thread;
}
  dequeue current thread if it was queued;   
  
release:
>update synchronization state;
if (state may permit a blocked thread to acquire)
unblock one or more queued threads;   

可以看出acquire就是尝试从队列中pop一个线程，release就是更新同步状态，释放资源给其他线程。   

AQS在实现以上操作，需要包含以下三个基本组件：   
> • Atomically managing synchronization state
• Blocking and unblocking threads
• Maintaining queues   

这个三个组件并不是绝对独立的，而是彼此相关。例如，队列中的节点信息需要与是否需要unblocking相一致。   
>The central design decision in the synchronizer framework was
to choose a concrete implementation of each of these three
components, while still permitting a wide range of options in
how they are used. This intentionally limits the range of
applicability, but provides efficient enough support that there is
practically never a reason not to use the framework (and instead
build synchronizers from scratch) in those cases where it does
apply.   

### Synchronization State   

刚刚说了，state就是一个32位的int，它有三个方法：   
1. getState
2. setState
3. compareAnsSetState

当时借助于jsr-133提供的新的java memory model对volatile的实现，以及底层compare-and-swap或load-linked/store-conditional指令实现的原子操作：compareAndSetState。   

关于java memory model，这里先不展开讨论。简单说来就是，在jsr133之前，volatile很鸡肋，并不能保证在compiler、cpu执行、内存读写时，由于reorder或者barrier存在的情况所导致的读写不一致的情况。在jsr133之后，就保证了在此期间不会reorder和正确的barrier。   

另外为什么使用32位保存状态，而不是64位的long？因为java一直吹逼跨平台，然而不是所有平台都支持64位的原子操作，如果底层不支持64位的原子操作，就需要用lock实现。简单说来，就是当时用64位还不是很靠谱，用32位还是可以接受的。在当时，concurrent包里只有CyclicBarrier需要更多bits来存储状态，所以这里就使用Lock代替了。现在，就我所知，FutureTask也不再使用AQS了，详见JDK。   

### Blocking   

直到JSR166，还没有基于内置monitors，通过创建synchronizer的，block或unblock线程的Java API。唯一看似可用的是通过：Thread.suspend/Thread.resume。然而这里会有一个无法解决的竞争问题：如果一个unblocking线程，这个线程在blocking线程执行suspend之前调用了resume方法，那么resume操作将不会生效。   

后来java jdk中就引入了LockSupport，这个class中的方法解决了上述问题。LockSupport.park阻塞当前线程，除非或直到LockSupport.unpark被处理。   

### Queues   

AQS的核心就是这个了，如何维护阻塞的线程——FIFO queues。数据结构选用CLH，具体啥是CLH我就不知道了，那篇论文我没有看。之所以选择CLH是因为，用它实现取消和超时操作更加简单。如果你阅读过很多java多线程相关的材料，或是有一定的多线程开发经验，你自然会知道，处理超时和取消是有多么重要，所以作为基础框架，如何支持处理超时和取消应该AQS首先需要思考的。   

一个重点是，CLH queue不像大多数的queue，因为它的进队和出队操作是作为lock来使用的。   

一个新节点，这里称为**node**，进队的原子操作：   

```java
do {
	pred = tail;
} while (!tail.compareAndSet(pred, node))
```

每个节点的释放状态都保存在其前驱节点中。所谓的spinlock中的spin就想这样子：    

```java
while (pred.status != RELEASED); //spin
```

出队操作很简单，只要把**node**指向头就好了。这样**node**就会获取锁
```java
head = node;

```

CLH lock的优点包括：进队出队快、lock-free、obstruction free（尽管有竞争，但是总会有一个线程在竞争中胜出。）通过观察head和tail是否相同就知道是否有线程在等待。

AQS在实现中用尽了原子操作，代码很难看懂。思路就是，如果有多个请求，并发的执行一段代码，原子操作要求只有符合目标预期的一个请求会真正执行原子操作，否则应该重新进入spin中。所以，编写原子操作的代码不能简单的考虑当前的可能性，应该把所有的可能性都考虑进去。

需要注意的是，一个节点的状态信息（status），其实是保存在它的前驱中的！获取锁和释放锁对应的就是进队和出队。需要注意的是，默认AQS是不公平的。这意味着，并不是head会第一个出队。这是出于效率的考虑。试想，一开始没有资源，但是当一个线程在acquire的时候，突然有资源被释放出来，而这个线程尚未进队列。这时候，这个线程会跳过进队操作，直接acquire。这样比进队列仔出队列效率要高。同样道理，对应notifyAll也是同样道理，不公平的效率更高。

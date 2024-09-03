---
layout: post,
title: Memory model
date: 2016-11-17
tags:
- java
- memory model
---

内存模型是一个相对底层的概念，对于很多编程语言来说，内存模型是对程序员屏蔽的概念。但是如果是涉及并发编程的话，内存模型是一个逃不开的知识点。本文是对自己理解的Java Memory Model的知识梳理。   

# What is reordering?

你所写的java代码会被compiler、JIT、cache、instructions、CPU执行时重排序。   

>Source order – how you write the code
– Naïve expectation that everything happens exactly as written
• Program order – generated machine code presented to the CPU
– “Any resemblance to source order is purely coincidental”
– It’s all about performance! As long as you can’t tell the difference
• Execution order – how the hardware actually executes the code
– Speculative execution, instruction reordering, caching, pipeline stalls
– It’s all about performance! As long as you can’t tell the difference
• Observation order – how things appear to happen for a given observer
– Different observers can see things happen in a different order

为啥要重排序呢？简单来说就是为了更好的performance。   

若是写单线程程序，reorder不是问题。若是多线程环境下，就会出现不同线程执行同一代码块时，所访问的变量的顺序不同。这时就会发生不一致的情况。内存模型就是为解决这个问题应运而生的。   

CPU发展到现在提升单核的运算性能已经很难了，只能靠在芯片上加更多的core，通过提高并发来提高性能。若是执行的代码没有Data Race，自然也不需要考虑这个问题。   

# Java Memory Model   

Java Memory Model为程序中的操作顺序定义一种__Happens-Before__的偏序关系，包括对变量的读写操作、监视器的枷锁和释放操作、以及线程的启动和合并操作。Happens-Before是一个死板的定义。这里举例来说。   

### synchronization under the hood

> Synchronization has several aspects. The most well-understood is mutual exclusion -- only one thread can hold a monitor at once, so synchronizing on a monitor means that once one thread enters a synchronized block protected by a monitor, no other thread can enter a block protected by that monitor until the first thread exits the synchronized block.
>
>But there is more to synchronization than mutual exclusion. Synchronization ensures that memory writes by a thread before or during a synchronized block are made visible in a predictable manner to other threads which synchronize on the same monitor. After we exit a synchronized block, we release the monitor, which has the effect of flushing the cache to main memory, so that writes made by this thread can be visible to other threads. Before we can enter a synchronized block, we acquire the monitor, which has the effect of invalidating the local processor cache so that variables will be reloaded from main memory. We will then be able to see all of the writes made visible by the previous release.

**synchronized**语义上是同步，但是从实现角度看其实就是一个**monitor**。这个monitor保证同一时间只有一个线程能进入这块代码。不仅如此，当一个线程**acquire** monitor的时候，保证当进入该代码块时，会向main memory读取所需的内容，而不是从CPU本地的cache或寄存器读。当该线程**release** monitor的时候，保证将发生变化的部分写回main memory。这就是**synchronized**如何保证一致性。从这里还可以看出，在synchronized block内访问的全局变量不需要是**volatile**。

### volatile under the hood

>Volatile fields are special fields which are used for communicating state between threads. Each read of a volatile will see the last write to that volatile by any thread; in effect, they are designated by the programmer as fields for which **it is never acceptable to see a "stale" value as a result of caching or reordering**. The compiler and runtime are **prohibited from allocating them in registers**. They must also ensure that after they are written, they are flushed out of the cache to main memory, so they can immediately become visible to other threads. Similarly, before a volatile field is read, the cache must be invalidated so that the value in main memory, not the local processor cache, is the one seen. There are also additional restrictions on reordering accesses to volatile variables.

在JSR133之前，volatile不能正常工作。那时的内存模型保证volatile变量彼此不会被重排序，但是会跟non-volatile变量一起被重排序。   

### final field in constructor

在construcot中初始化final field时，final field不会被重排序，该field所指向的变量初始化也不会被重排序。

```java
class FinalFieldExample { 
    final int x;
    final Map map;
    int y;
    static FinalFieldExample f;

    public FinalFieldExample() {
        x = 3; // no reorder
        y = 4;
        map = new HashMap(); //no reorder
        map.put("test", 1); //no reorder
        map.put("test", 2); //no reorder
    } 

    static void writer() {
        f = new FinalFieldExample();
    } 

    static void reader() {
        if (f != null) {
            int i = f.x;  // guaranteed to see 3  
            int j = f.y;  // could see 0
        } 
    } 
}
```

# Conclusion

编写多线程代码时，围绕的核心是对产生的data race问题，需要按照happens-before的顺序，使用一对acquire和release，对synchronized block或volatile field正确访问和发布。

当使用constructor进行初始化的时候，合理使用final修饰符。

JMM已经制定好了游戏规则，我们写代码的时候只需要按照规则来就行了。JMM对程序员的意义就在于理解规则。

# Referrences

1. [jsr133-faq](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)
2. [Memory barriers/fences](https://mechanical-sympathy.blogspot.jp/2011/07/memory-barriersfences.html)
3. [jmm handbook](http://g.oswego.edu/dl/jmm/cookbook.html)
4. [final feild in constructor](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.5)


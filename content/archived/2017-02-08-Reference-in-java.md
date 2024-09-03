layout: post
title: Java中的引用类型
date: 2017-02-08
tag:
- java
- reference
---

最近在写一个Java实现的cache，就是一个guava cache的轮子。   

之前没有刻意研究guava cache，这几天重温guava源码的时候，发现guava cache中继承了SoftReference, WeakReference, PhantomReference，里面还有实现了的queue，这些reference会被enqueue，好奇心因此产生。   

看到上述代码的时候才发现自己不知道如何在java中使用这些reference，于是研究了一番。大家经常在面试、笔试中碰到的reference的问题大都是问：“Java中有几种reference？分别是什么？”但是很少问它们究竟都是用来干什么的？如何使用？   

[java.lang.ref](https://docs.oracle.com/javase/8/docs/api/java/lang/ref/package-summary.html)上有详细的解释，但是不是很通俗。由于我是想自己写一个cache，而SoftReference非常适合开发cache程序，我就特别关注这个问题。总的来说，这些reference都是跟垃圾回收器相关，会针对不同的reference类型，产生不同的回收策略。   

>Soft references are for implementing memory-sensitive caches, weak references are for implementing canonicalizing mappings that do not prevent their keys (or values) from being reclaimed, and phantom references are for scheduling pre-mortem cleanup actions in a more flexible way than is possible with the Java finalization mechanism.   

为什么说SoftReference适合做cache？因为当你为一个普通对象创建一个SoftReverence，即使当这个对象不再拥有任何其他对象的强连接，垃圾收集器也可能不会立即将SoftReference回收，而是等到剩余内存达到阀值了，不得不进行垃圾回收的时候才将SoftReference引用的对象回收。因此，在构建cache程序时，这里就可以使用SoftReference构成多级cache架构。第一层是你的cache数据结构，可能是一个自己实现的Map，第二层是SoftReference。

### 对比SoftReference、WeakReference和PhantomReference   

```java
ReferenceQueue<Foo> fooQueue = new ReferenceQueue<>();
Foo foo = new Foo();
Reference<Foo> reference = new WeakReference<>(foo, fooQueue);
foo = null;
System.gc();
System.out.println(reference.get() == null); //true
```

在JVM开始gc的时候，由于foo没有任何连接对象，WeakRefeence会直接被GC回收。   

```java
ReferenceQueue<Foo> fooQueue = new ReferenceQueue<>();
Foo foo = new Foo();
Reference<Foo> reference = new SoftReference<>(foo, fooQueue);
foo = null;
System.gc();
System.out.println(reference.get() == null); //false
```

而当使用SoftReference时，即使foo没有任何强连接，在发生GC的时候，由于内存尚且充足，SoftReference不会被立即回收，而是在将来内存不够用的时候再回收。   

```java
ReferenceQueue<Foo> fooQueue = new ReferenceQueue<>();
Foo foo = new Foo();
Reference<Foo> reference = new PhantomReference<>(foo, fooQueue);
foo = null;
System.out.println(reference.get() == null); //true
```

PhantomReference作为最弱的引用对象，当一个对象只被一个PhantomReference引用时，不管是否发生GC，这个对象都会被回收。   

有趣的是，以上对于PhantomReference的解释是一个错觉。因为PhantomReference.get()永远返回**null**。JDK文档解释如下：   

```java
/**
 * Returns this reference object's referent.  Because the referent of a
 * phantom reference is always inaccessible, this method always returns
 * <code>null</code>.
 *
 * @return  <code>null</code>
 */
public T get() {
    return null;
}
```

phantom reference意味着所引用的对象已经不存在了，就等着被GC了，一旦被JVM回收，JVM会主动调用enqueue()，将phantom reference加到ReferenceQueue中。phantom reference存在的目的是为了配合finalization。我们可以在java中重写Object的finalize()方法，但是finalize方法不应该随便使用，通常我们不需要finalize方法。phantom reference的存在补充了finalize的功能。phantom reference可以用来处理对象**死前**的cleanup操作。   

```java
Foo foo = new Foo(1);
Reference<Foo> reference = new PhantomReference<>(foo, fooQueue);
foo = null;
while (true) {
	System.out.println(reference.isEnqueued());
	if (reference.isEnqueued()) {
    	break;
    }
}
```

当foo只被phantom reference引用时，foo已经死了，当GC触发回收foo对象时，JVM会将foo的phantom reference加到fooQueue中。即，当发现phantom reference在queue中时，就意味着foo对象已经不在内存中了。   

### 举个例子

当需要load一张大图片到内存中，我们希望当一张图片被GC完全回收后再load下一张，这时候就可以利用phantom reference来实现了。   

>Similarly we can use Phantom References to implement a Connection Pool. We can easily gain control over the number of open connections, and can block until one becomes available.

### ReferenceQueue是个什么鬼？

如果单从文档看，我完全没有看出ReferenceQueue是干什么用的。。。可能是因为这个类的调用者不仅仅包含编写代码的程序员，还包括JVM。可以理解为将已经被GC回收的对象加到queue中。我们可以自己控制，也可以由JVM调用完成，具体情况就要看如何实现了。

### FinalReference

这个应该是最抽象的一类reference，@你假笨同学给出了更加详尽的解释，在这里通俗的说明一下。

什么样的对象会成为FinalReference呢？一个重写了Object下的finalize()且finalize()方法体不为空的对象！

我们知道finalize()类似于析构函数，但是java中有垃圾回收，我们不应该手动触发finalize()，所以这个方法的调用也确实是被JVM调用的。看源码会发现，FinalRefernce是包级私有的，同包下有一个声明为final的子类：Finalizer。

当一个实现了finalize方法的对象通过调用构造方法初始化时，JVM就会通过调用Finalizer类中的register方法来注册。Finalizier中还会初始化一个FinalizerThread，用于通过调用runFinalizer()执行销毁操作。

什么情况下我们需要重写finalize方法呢？可能我们永远不需要。。。可以参考FileInputStream的实现。finalize可以用于执行close操作，但是由于执行销毁的线程FinalizerThread优先级比较低，所以不能保证会及时销毁，看起来还是很鸡肋的。。。

# Reference
1. [Difference between WeakReference vs SoftReference vs PhantomReference vs Strong reference in Java](https://www.javacodegeeks.com/2014/03/difference-between-weakreference-vs-softreference-vs-phantomreference-vs-strong-reference-in-java.html)
2. [What is a Soft Reference Cache Anyway?](https://www.ortussolutions.com/blog/tip-of-the-week-what-is-a-soft-reference-cache-anyway)
3. [Finalization and Phantom References](https://dzone.com/articles/finalization-and-phantom)
4. [JVM源码分析之FinalReference完全解读](http://lovestblog.cn/blog/2015/07/09/final-reference/)
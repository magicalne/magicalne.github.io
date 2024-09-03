---
layout: post
title: 关于java中的assert
date: 2016-11-04
categories:
- java
---

自从最近才发现，我一直在用错误的方式使用assert！这一切都源于自己的想当然没有去阅读[java doc](https://docs.oracle.com/javase/8/docs/technotes/guides/language/assert.html)！   

原来我使用assert的方式就是方法的输入参数校验，这是错误的。特别对于public方法。因为assert校验失败之后抛出AssertError，而不知道到底是什么原因导致了校验失败，所以参数校验还是应该使用IllegalArgumentException。   

>Do not use assertions for argument checking in public methods.   
>
Argument checking is typically part of the published specifications (or contract) of a method, and these specifications must be obeyed whether assertions are enabled or disabled. Another problem with using assertions for argument checking is that erroneous arguments should result in an appropriate runtime exception (such as IllegalArgumentException, IndexOutOfBoundsException, or NullPointerException). An assertion failure will not throw an appropriate exception.   

另外一个更为重要的是：assert最为一项jvm支持的功能，默认情况是关闭的。只有在java option中指定**-enableassertions**或者是**-es**才可以   

那么assert应该怎么用呢？   
1. 不要用做public方法的输入参数检查。
2. 关于throw什么exception应该自己在程序中控制，而不是依赖assert。
3. 使用assert验证算法执行中的不变量前置条件和后置条件，而**不是验证数据的正确性**。

第三条说的太宽泛了，具体的可以参考java docs中更加详细的说明。   

然而我发现和我一样犯这种错误的人不在少数，很多开源框架中的源码也有很多违背使用assert的地方，特别是在public方法中用assert校验输入参数这条。

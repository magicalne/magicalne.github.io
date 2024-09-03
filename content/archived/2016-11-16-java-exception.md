---
title: 关于java异常
layout: post
date: 2016-11-16
categories:
- java
- exception
---

写代码的时候一直使用checkstyle和findbug，checkstyle和findbug可以帮助你写出标准规范的代码。特别是跟intellij插件结合之后，你的缩进、格式、甚至是异常控制都会被规范化。比如之前的项目中，对于异常的管理很不严格，大家都是所以的throw Exception和catch Exception。所以有一天我就抽时间将这些Exception都改成了具体的某个异常。   

但是我发现，有些地方catch Exception甚至是catch Throwable是很必要的。   

比如rabbitmq处理rpc返回的地方，如果发送的rpc请求在worker端发生异常，这个异常可能是任何异常，甚至可能是一个Error，所以rabbitmq这里其实会throw Throwable。当然，这里其实是spring amqp抛出的。但是在处理这类异常的时候，还是不应该让Throwable流到上层，因为客户端只需要知道这里出现异常应该怎么办，而不是关心整个异常链。所以应该这样：    

```java
try {
	xxx
} catch (Throwable t) {
	Throw new SomeRPCException(t, "blablabla");
}
```

另外在试图从异常中恢复，特别是在多线程中，尝试因为抛出的异常终止或恢复线程，也需要catch Exception／Throwable。

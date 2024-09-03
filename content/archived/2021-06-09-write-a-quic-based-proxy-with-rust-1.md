---
title: 使用rust写一个基于QUIC的proxy[1]
layout: post
date: 2021-06-09
tag:
- rust
- tokio
- proxy
- future

---

[magicalane](https://github.com/magicalne/magicalane)最新版本v0.1.0已经支持socksv5，实测下来效果还不错呢。

当第一次开始测试时，发现不能打满带宽，这让我对quic产生了怀疑。腾讯云lighthouse带宽是30Mbps，但是magicalane单链接只能到大约8kb/s。

首先quic这个协议本身的[性能](https://github.com/quicwg/base-drafts/wiki/21st-Implementation-Draft#perf-features-tested)确实不如tcp，所以没能打满带宽也算是意料之中。但是差的有点大阿。

首先怀疑的是quinn-quic实现问题，看过一陀一陀的代码也没有发现问题。回头又一想，这个**8kb**怎么这么眼熟呢。然后看到自己代码里把proxy的buffer size设定为8kb了。

原来瓶颈是我的hardcode0.0

我能接触到的服务器中，能给出的最大带宽也就30Mpbs。所以我就依照这个30Mpbs，然后改为30 * 128 * 1024 = 3932160，然后就起飞了。

单链接看youtube 8K视频没问题，就是自己的浏览器有点卡，网络是不卡的。带宽基本打满了，cpu和内存使用率也很低。

联想到v2ray，是不是也有buffer size的参数可以配置？一顿搜索之后，果不其然，配置里也有一个bufferSize。所以继续测试了相同buffer size的v2ray的效果。结果是v2ray效果更好。

当然最后这点带宽不是很重要了，使用quic最重要的原因是想解决head of line blocking。接下来会新增一个tcp+tls的proxy实现，这样才能更好的对比性能。

同时再考虑如何综合quic和tcp的优势。比如当网络不稳定，出现了～10%丢包，这时候切换成quic；当网络稳定了，再切回tcp。

# 手写的**未来**

手写Future真的很难。[proxy](https://github.com/magicalne/socks5lib/blob/main/src/proxy.rs)的实现直接port了[linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy/blob/main/linkerd/duplex/src/lib.rs)。

手写Future所带来最大的好处就是充分使用范型，可以写抽象的代码。虽然有async-trait这种crate，bin用用可以的，但是lib还是乖乖手写吧。因为lib在设计时候可以在最后阶段交付一个非常generic的Future，还是非常人体工学的。

手写Future的时候，需要反复利用Context wake task的机制。因为在Future实现中，通常会多次调用poll_like方法，而poll_like方法需要传入cx。这时候，如果有多个poll_like方法返回pending，当某一个方法能够产生progress时，只有最近调用的Future会被调用。

也就是说，向poll_like方法传入cx，只是**订阅**了一个`topic`，当能够产生progress时，这个`topic`不会直接触发回调机制。Future只能通过循环状态机的方式工作。

所以呢，当某个poll_like方法返回pending，我们能够明确知道，当Future产生新的progress，我们会继续回到这块代码，因为我们之前已经订阅了这个`topic`。这么说还是挺绕的。简单的状态机很好写，一个enum搞定。复杂的就难了。。。

# pin

再一个难点就是`pin`。回想过去学习计算机编成相关的概念，貌似没有哪个概念再比pin复杂了吧？当初第一次接触pin，没有什么特殊感觉，以为自己理解了。不就是把一个struct pin到stack/memory么，这样Future调用的时候，即使被其他的cpu core调用，物理内存地址也不会发生修改，这样就能保证正确性。

后来又看了看Unpin，!Unpin，Pin<Box<T>>，再加上deref、structural，蒙了。。。

Pin相关的文档不知道读了多少遍了。对我造成最大困扰的地方就是无法直接查看async fn生成的代码，看编译后的汇编没啥效果。

最后通过实践，有两个特性尤为重要：

1. 只有当struct所有成员都是Unpin，那这个struct才是Unpin，这是编译器默认的。否则只能通过实现Unpin trait来确保自己是Unpin，幸运的是，实现Unpin是安全的；
2. 如果一个T是Unpin，那么Pin<&mut T>与&mut T等价，可以随意deref。

之所以上述两个特性很重要，是因为Pin的设计之初是为了async生成代码时，会生成自引用代码，所以自引用的成员需要声明为Pin。但是平常手写Future，基本不涉及自引用变量，所以这时候的问题就转变为如何去除Pin。依照上述两个特性，大部分的struct都可以是Unpin的。

再接下来就是structual project。我用下来，最需要Pin的时候，是当我需要poll一个async fn。也就是类似：

```rust
struct MyFut<O> {
    inner: Pin<Box<dyn Future<Output = Result<O> + Send + 'static>>>
}
```

其他时候可以用pin-project，写起来非常人体工学。当然也可以不用。不用pin-project的好处，就是可以通过deref调用&mut self声明的poll_like方法。

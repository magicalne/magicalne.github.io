---
title: 使用rust写一个基于QUIC的proxy[0]
layout: post
date: 2020-08-06
tag:
- rust
- quinn
- hyper
- tokio
- proxy

---

# 简介

## QUIC

From wikipedia:

> QUIC (pronounced "quick") is a general-purpose[1] transport layer[2] network protocol initially designed by Jim Roskind at Google,[3] implemented, and deployed in 2012,[4] announced publicly in 2013 as experimentation broadened,[5][6][7] and described to the IETF.[8] While still an Internet Draft, QUIC is used by more than half of all connections from the Chrome web browser to Google's servers.[9] Microsoft Edge[10] and Firefox[11] support it, even if not enabled by default, as does Safari Technology Preview.[12]

简单说，QUIC是一个基于UDP的可靠链接，协议定义需要tls，所以QUIC本身是可靠且安全的。因为是基于UDP，所以自然继承了UDP的所有优点。可以做到0RTT或0.5RTT。我看到国内的头条也有[QUIC的应用](https://www.infoq.cn/article/OH79WEaK7Z3S2XaVO*BV)了。B站的视频加载API已经是H3了！不清楚直播的上传API是不是基于QUIC或者H3，直播观看还是http1.1。

因为UDP是基于connection id的，所以当client从wifi切到4G是不会断网的。相对于TCP，QUIC最大的优点：彻底解决了[head of line blocking](https://http3-explained.haxx.se/en/why-quic/why-tcphol)。

QUIC也是google发起的，前身就是SPDY。有人说SPDY失败的原因在于过去几十年的硬件优化都是围绕TCP展开的，而SPDY的发展需要硬件厂商放弃过去的投资，重新投入。现在看来QUIC也有这方面的局限。据说UDP流量也会被QOS。在带宽有限的情况下，流量突然增大，TCP是网络提供商重点关注的，那自然UDP就是可以舍弃的。但是这都是YY，我没有实测。如果说打游戏算实测的话，那UDP国内情况还好吧，出口到国外估计会QOS。这也就是为什么有些代理会把UDP包成TCP再发。

另外一点，HTTP3就是基于QUIC的，这也算是个大背书吧？我看taobao的首页中，cdn资源99%都是H2的。但愿H3不会远吧。

### QUIC实现

QUIC的实现还是挺多的。rust这边主要有三个实现：

1. [cloudflare quiche](https://github.com/cloudflare/quiche)
2. [mozilla neqo](https://github.com/mozilla/neqo)
3. [quinn](https://github.com/djc/quinn)

其中quiche在cloudflare生产环境跑起来了，neqo也算是生产级别吧。二者都很底层，quiche在tls这层用了google的[boringssl](https://boringssl.googlesource.com/boringssl/)，需要自己编译，这个东西我下载不下来。。。。neqo是自己实现的crypto。quinn使用rustls。印象中rustls的inbound和outbound session是不能并发访问的。看star的话，无疑quiche是rust首选。虽然quiche是一个实验型的项目，但是quinn实现到了tokio这层，所以写起来是最方便的，我还是决定先用quinn试试水。

## 代理

代理实现太多了，看了酸酸乳、V射线和某木马的源码，也没看懂...可怜的我c、c艹、go一个都不会。但是基本实现我还是看明白了的。

1. 代理的客户端负责接受请求，请求类型只能是UDP、TCP；
2. 然后把接受的请求发送给代理服务器端，内容包括：需要代理的uri和整个request buf；
3. 代理服务器端接受到uri和reqeust buf，需要与远端的uri建立链接，然后再把整个buf发送过去；
4. 远端服务器把response发回给代理服务器；
5. 代理服务器把接受到的response发回给代理客户端；
6. 代理客户端把response发送给请求的调用方。

如果代理的实现能让请求发送端看不出有代理的**痕迹**，那就是成功的透明代理了。这里会牵扯出一系列的问题，核心问题就是怎么维护连接。作为第一个版本，我就打算写个最简单的，以能跑起来为目的。客户端是一个http proxy + QUIC client，服务端是QUIC server + TcpStream。

# HTTP Proxy

HTTP Proxy有一个特别的method：**connect**

```http
CONNECT www.example.com:443 HTTP/1.1
```

意味着，client请求代理打开一个proxy tunnel，代理回复OK，client会把真正的请求发送给代理。我在hyper的example里找到了一个[HTTP Proxy例子](https://github.com/hyperium/hyper/blob/3de81c822e6ac5a5b0640059f53838d0906f68c4/examples/http_proxy.rs)。只要把QUIC client的实现加上去，客户端就算写完了。

这个HTTP Proxy的实现重点有两个

## 1. 在接到connect之后创建一个tokio task，并把与client的链接传进去，然后直接return OK。

```rust
 if let Some(addr) = host_addr(req.uri()) {
    tokio::task::spawn(async move {
        match req.into_body().on_upgrade().await {
            Ok(upgraded) => {
                if let Err(e) = tunnel(upgraded, addr).await {
                    eprintln!("server io error: {}", e);
                };
            }
            Err(e) => eprintln!("upgrade error: {}", e),
        }
    });

    Ok(Response::new(Body::empty()))
}
```

这样连接是不会丢失的。具体的代理逻辑放到了异步task中执行，当前的请求response直接返回给客户。

## 2. 使用tokio::io::copy和try_join执行流量代理

```rust
// Create a TCP connection to host:port, build a tunnel between the connection and
// the upgraded connection
async fn tunnel(upgraded: Upgraded, addr: SocketAddr) -> std::io::Result<()> {
    // Connect to remote server
    let mut server = TcpStream::connect(addr).await?;

    // Proxying data
    let amounts = {
        let (mut server_rd, mut server_wr) = server.split();
        let (mut client_rd, mut client_wr) = tokio::io::split(upgraded);

        let client_to_server = tokio::io::copy(&mut client_rd, &mut server_wr);
        let server_to_client = tokio::io::copy(&mut server_rd, &mut client_wr);

        try_join(client_to_server, server_to_client).await
    };

    // Print message when done
    match amounts {
        Ok((from_client, from_server)) => {
            println!(
                "client wrote {} bytes and received {} bytes",
                from_client, from_server
            );
        }
        Err(e) => {
            println!("tunnel error: {}", e);
        }
    };
    Ok(())
}
```

tokio::io::copy在读到异常或读到0个字节会退出，也就是表示这次连接结束了。task return之后，两个连接连同tokio task一同被drop了。

我一开始以为copy这个方法会无限loop。其实是只读写一次的。所以每次请求都要开一个proxy tunnel。

# 后记

整个代理分别实现了client、server和protocol。没有特别复杂的东西，quinn用起来还挺简单的。唯一遇到的问题就是tokio::async_read_ext::read_buf这个方法与std的read逻辑不一样。tokio是根据传的buffer size来决定读多少。比如一个http的content response是同步的，我应该一次read就读完整个response。但是，如果使用tokio read，传入的buffer: 

```rust
let buf: Vec<u8> = Vec::with_capacity(1024);
```

那就只能读1024个字节。即便response是1025，那最后一个是读不到的。

随即我就想抄代码了。。。

开始翻越hyper的源码，想看看hyper是怎么实现读response的。然后就是一连串的惊呼...mum...

如果我没看漏，hyper从头到尾是没有直接调用read、write方法的。严格说，hyper从头到尾没有调用过async方法。hyper所有的处理逻辑都放在了poll里面。通过调用poll、实现自己的poll和状态机，方便了用户只在调用入口处使用**await**。非常简洁！看完hyper的源码，我算是入门future了。整个代码仓库基本没啥注释，看的我晕晕的。Hyper的作者看起来年纪也不大，rust在http这块的版图基本上都是这哥们贡献的。之前在mozilla，今年5月去了AWS。粉了粉了~

# 总结

史上最简单的基于QUIC的透明代理算是完工了。可以方便的代理http1请求，没试过http2。性能就不谈了，完全发挥不出rust的威力。

接下来会优化client和server管理连接的实现；以及，增加客户端代理类型：socks5。

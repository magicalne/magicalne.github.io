---
title: 在netty中同时使用http proxy和websocket带来的问题
layout: post
date: 2019-04-16
tag:
- netty

---

最近在用netty开发自己的交易系统。http那块没啥问题，很流畅。在开发基于websocket的market data部分出现了问题。表现为可以与server建立http连接，在upgrade的时候也成功，而且也接收到了server发送过来的数据，但是不能向server发送数据。msg已经被encode成WebSocketFrame了，但是没有发送到server。

因为远端server被墙了，所以只能加代理。init的时候在最前面加上HttpProxyHandler。handler pipeline看起来像这样：
> http proxy handler -> ssl handler -> http client codec -> http agg -> my handler

问题就出现在netty官方提供的WebSocketClientHandshaker。[具体issue](https://github.com/netty/netty/issues/5201)

在websocket upgrade的时候（也就是websocket handshake），需要加入websocket encoder和decoder，然后删除http client codec。netty在这里没有处理存在http proxy的情况，默认删除第一个http codec。而且插入websocket encoder和decoder的位置也错了。

这就导致我之前遇到的问题，在消息encode好的情况下，无法通过http proxy handler发送出去。

我的解决办法是继承WebSocketClientHandshaker13，自己实现了handshake0和finishHandshake0，并在初始化pipelie的时候使用别名。

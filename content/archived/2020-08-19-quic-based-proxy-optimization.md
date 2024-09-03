---
title: 基于QUIC的透明代理客户端优化
layout: post
date: 2020-08-19
tag:
- rust
- proxy
- quinn

---

在基本功能完善之后，发现代理速度如龟。这跟理论不贴合阿，先从client入手。

第一个问题非常明显，client的udp连接没有做到复用。QUIC是支持单udp socket上建立多connection。而我在实现的时候是把每个tcp连接代理出去的时候就建立一个udp连接。这里改成向future传入&Endpoint就好了。

但是改完之后还是好慢，没有任何改善。

我开始打开trace log，观察twitter首页的加载。一段在http1.1的传过来js，只有～140kb，居然要用10多秒。而log显示每次recv都只有30-40字节，都不是kb...本地代理出去的时间很快，不到1ms，关键就是waiting(TTFB)的时间有点久了，下载速度是肉眼可见的慢，但是还是在下载的，没有断掉。

quinn文档有说到，遇到延迟特别大的时候可以考虑调整SO_SNDBUF and SO_RCVBUF，我一直以为这两个参数值的是我的应用层的buffer size。后来才想到这个可能是操作系统层面的参数。

在我的fedora运行：

```shell
$ sysctl net.core.rmem_max
net.core.rmem_max = 212992
$ sysctl net.core.rmem_default
net.core.rmem_default = 212992
```

也就是说最大不过208kb？改成25M试试:

```shell
$ sudo sysctl -w net.core.rmem_max=26214400
net.core.rmem_max = 26214400
$ sudo sysctl -w net.core.rmem_default=26214400
net.core.rmem_default = 26214400
```

速度飞起，心疼流量...

目测+心算，http1.1下应该是**ray的10倍吧。youtube视频4k随便拖拽～～

之所以想自己写代理的原因，就是在看youtube的时候突然卡掉了，telnet却是通的。等youtube缓冲半天，进度条还是纹丝不动，看看telnet还是通的！

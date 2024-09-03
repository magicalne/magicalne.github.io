---
title: 在rust中处理证书时遇到的问题
layout: post
date: 2020-08-14
tag:
- rust
- proxy
- tls
- let's encrypt

---

基于QUIC的透明代理走到上线环节了。在测试过程中遇到了关于TLS证书的问题，这里记录一下。

# 背景

在google cloud上有一台instance，因为已经安装了v2ray+caddy，所有tls已经有了。首先要找到server上证书的地址。

一顿find之后，发现了两个文件server.crt和server.key。我就想当然的把这两个文件当作cert和key传给server端了。

结果就报错了。

# BadDER

这个错太简写了。。。我发现是key出的问题。我对证书的各个版本和格式不是很了解，这里面有点复杂，我只是知道个大概。[这篇文章解释的很清楚。](https://support.ssl.com/Knowledgebase/Article/View/19/0/der-vs-crt-vs-cer-vs-pem-certificates-and-how-to-convert-them)

我打开.key文件开到头尾是这样的：

```shell
-----BEGIN EC PRIVATE KEY-----

----END EC PRIVATE KEY-----
```

## ec private key是个什么鬼？

[解释在这里。](https://wiki.openssl.org/index.php/Command_Line_Elliptic_Curve_Operations)

github上找到了相关讨论，源码这里不支持ec key，所以需要把ec转为.pem文件。

```shell
openssl pkcs8 -topk8 -nocrypt -in server.key -out server.key.pem
```

重试一下，报了另一个错...

# UnknownIssuer

通过对比client和server的log，确定这个错是从client的webkpi报出来的。确定是在client接受到server发回来的tls public key之后报的错。这个阶段发生在quic 1-rtt阶段。

所以server这里是没问题了。

再看这个错，字面意思就是Unknown Issuer...

## 什么是Issuer呢？

这要说说TLS/SSL的工作原理。简单说，server有一套cert和key，用来向client保证:"咱俩的连接是安全的"。那凭什么有了cert和key，连接就安全了呢？那是因为cert和key是由一个有公信力的组织发给server的。比如，这里的let's encrypt就是一个有公信力的组织，internet上都信任这个组织。全世界的公共发证机关就那么些，chrome、mozilla、操作系统会把各个安全的机关记录下来。这个叫ca root store。

这些发证机关叫Authority。可以在chrome打开**chrome://settings/certificates**，就能看到了。

Issuer就是发证机关。

## Issuer去哪了呢？

首先，caddy是用的[Let's encrypt](https://letsencrypt.org/)。难道这个背书在我的fedora32上没有嘛？我第一次搜还真没有。


```shell
sudo dnf install ca-certificates
```

```shell
awk -v cmd='openssl x509 -noout -subject' '
    /BEGIN/{close(cmd)};{print | cmd}' < /etc/ssl/certs/ca-bundle.crt | grep "lets"
```

这时候搜，确实能搜到let's encrypt。但是还是不行。

这里我想是不是代码里面根本没有去读root store？

看了半天代码，确定root store是能读到的，自己还写了个demo，用rustls_native_certs::load_native_certs()直接就拿到了。

## 真相只有一个。。。

.crt有问题

第一次尝试把.crt转为.pem:

```shell
openssl x509 -in server.crt -out server.cert.pem -outform PEM
```

还是不行。。。

再看了一遍文档：
>.CRT = The CRT extension is used for certificates. The certificates may be encoded as binary DER or **as ASCII PEM**. The CER and CRT extensions are nearly synonymous.  Most common among *nix systems

我看.crt的内容不是二进制的，所以，文件本身就是pem...我直接改后缀为.pem...成功了！

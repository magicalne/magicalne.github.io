---
title: 升级到fedora 33的问题列表
layout: post
date: 2020-12-08
tag:
- fedora
---

升级到fedora 33，没有被btrfs摧残，yet。

第一个问题是拉代码发现“no permission”。搜到了[redhat bugzilla](https://bugzilla.redhat.com/show_bug.cgi?id=1890179)

>The server probably does not support SHA2 signatures. You'll have to switch to LEGACY policy.

```shell
update-crypto-policies --set LEGACY
```

无需重启，问题解决。

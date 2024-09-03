---
title: 如何为Fedora删除kernel
layout: post
date: 2020-01-15
tag:
- linux
- fedora

---

Linux各个发行版的升级一直以来都是一件头疼的事情。当年第一次用ubuntu的时候，一周重装了3次...真的是遇到问题只会重装。

不过linux的增量更新并且结合grub，在一定程度上降低了升级系统的风险。

昨天我将我的Fedora 30进行了升级，kernel从5.3.16-200.fc30.x86_64升级到了5.4.6。然后就不能开机了。。。讲道理，用了这么多年的linux，以前这种事还是经常发生的，但是近几年真的没啥印象。用户体验的质量还是有把控的。

网上搜了一下，发现这个版本出问题的人还不少呢。好在grub上可以选历史kernel，我就切回了5.3.16。然后我就在5.3.16-200.fc30.x86_64这个image上把之前安装的最新的kernel删掉了。

步骤很简单，首先确认当前的kernel版本：

```bash
uname -r
5.3.16-200.fc30.x86_64
```



然后看看最近安装的kernel：

```bash
dnf repoquery --installonly --latest-limit=1 -q    

kernel-0:5.3.16-200.fc30.x86_64
kernel-core-0:5.3.16-200.fc30.x86_64
kernel-devel-0:5.3.16-200.fc30.x86_64
kernel-modules-0:5.3.16-200.fc30.x86_64
kernel-modules-extra-0:5.3.16-200.fc30.x86_64
```

然后删除上述版本的kernel：

```bash
sudo dnf remove $(dnf repoquery --installonly --latest-limit=1 -q)
```

reboot之后，老版本的kernel就不复存在了。

真香！

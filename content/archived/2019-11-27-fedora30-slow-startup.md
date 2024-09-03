---
title: 如何解决Fedora启动慢的问题
layout: post
date: 2019-11-27
tag:
- Fedora

---

家里的台式机是Fedora30/Win10 dual OS，Win10启动是秒开，毕竟有SSD。但是Fedora30开机慢的要死。起码需要1分钟了。

现在开始解决。

```bash
systemd-analyze blame
```

首先查看启动慢的原因，我的机器有两项超过30秒
1. systemd-udev-settle
2. dnf-makecache.service

systemd-udev-settle是跟驱动有关的。我是N卡，Fedora30自带的驱动是不兼容的，我是后面进入terminal另装的驱动。不知道跟这个是不是有关系，估计是个bug。看网上说这个是可以通过mask跳过的。
所以：

```bash
systemctl mask systemd-udev-settle
```

dnf-makecache.service这个东西想不明白为什么要放在启动之前。看网上说确实是可以放在启动之后的，so：
```bash
sudo systemctl disable dnf-makecache.service
sudo systemctl disable dnf-makecache.timer
```

来测试一下效果吧。

reboot
.
.
.

```bash
systemd-analyze blame
```

```
Startup finished in 9.589s (firmware) + 1.413s (loader) + 1.353s (kernel) + 1.609s (initrd) + 8.030s (userspace) = 21.995s 
graphical.target reached after 3.070s in userspace
```

虽然有21秒，但是纯启动时间是能到秒的，满意。

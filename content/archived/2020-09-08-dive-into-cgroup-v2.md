---
title: cgroup V2
layout: post
date: 2020-09-08
tag:
- cgroup
- systemd
- linux

---

虽然docker一直有用，但是cgroup、namespace只停留在字面上见过的级别。最近上手rust，想用rust写一个oci的实现，发现rust下有一些cgroup的crate。进而发现cgroup v2已经release有一段时间了，crun、runc等等都已经支持上了。但是rust中没有cgroup v2对应的实现。所以，写一个？

看了下[文档](https://man7.org/linux/man-pages/man7/cgroups.7.html)，v1和v2的差距还是很大的。网上介绍版本差异的文章很多了，intel的这个[pdf](https://events.static.linuxfound.org/sites/events/files/slides/cgroup_and_namespaces.pdf)介绍的足够了。

我的Fedora32已经默认支持上了cgroup v2，所以我按照文档说明用rust写了写测试代码。从最简单的创建一个cgroup开始。我是按照文档中操作fs的方式实现的：

```shell
sudo mkdir /sys/fs/cgroup/mycgv2
```

这里，我陷入了深深的沉思...

# 为什么要sudo？

cgroup是可以mount的类型之一，创建一个新的cgroup必须要用mount，而mount是需要root权限的。而且/sys/fs/cgroup就是在root下的，这里的root是逃不掉的。

那就用已有的cgroup呗。我发现cgroup有一个user目录：**/sys/fs/cgroup/user.slice/user-1000.slice**

1000是linux为初始化创建用户的id，也就是“我”的id，这个目录下的user和group都是“我”，也就是说我可以直接操作这个目录下的cgroup！那就可以不需要root了！

那么问题来了...

# 这个user.slice/下面的目录是谁创建的？

这个问题不难搜，目录下的文件名结尾都是.scope/.slice/.service。一看就是systemd创建的。我在另一个目录下用mount重新创建了一个cgroup挂载，发现这个user.slice/目录依然是自动创建好了的。从这里可以看出，cgroup是被systemd管理的。虽然文档并没有说明，但是后来我在社区的邮件也找到了证据。而且不应该在根目录下创建cgroup，因为这里是systemd管理的。

那是不是说，与cgroup正确的交互姿势是使用systemd？而不是fs？

# 其他cgroup v2实现

这里参考了runc/crun等等项目的实现，基本把cgroup v2的实现都看过一遍了。发现其实fs和systemd的实现是都用的，但是runc在开启rootless模式下建议使用systemd，但不强制。

这里耗费了好多时间，cgroup的文档读了好多好多遍，就想从字里行间理解systemd存在的意义。因为我知道一开始systemd虽然也支持v1，但是kernel还有libcgroup的实现。而到现在，libcgroup应该没有v2对应的release，官方代言就是systemd。在做了几天功课之后，我认为需要systemd接管cgroup的原因还是在于rootless。

# cgroup.subtree_controller和cgroup.controllers

cgroup.subtree_control是可读写的，用来限制子目录下的cgroup能用什么controller，就是v2的权限控制。
cgroup.controllers是只读的，用来展示当前能用的controller都有什么。

默认在跟目录下，cgroup.controllers包含全集：

```shell
$ cat cgroup.controllers

cpuset cpu io memory hugetlb pids
```

但是默认cgroup.subtree_control只包含pids和memory。所以使用rootless，默认的所有子cgroup只能操作pids和memory。想要操作更多controllers，只能修改根目录下的cgroup.subtree_control。有两种方式：

1. root用户下，echo "+cpu +memory -io" > cgroup.subtree_control
2. 通过systemd，当然也需要root，修改配置文件并reload:

```shell
$ mkdir -p /etc/systemd/system/user@.service.d
$ cat > /etc/systemd/system/user@.service.d/delegate.conf << EOF
[Service]
Delegate=cpu cpuset io memory pids
EOF
$ systemctl daemon-reload
```

两种方式都需要root权限，这可不妙阿。假设系统初始化与cgroup的管理分割开来，只需要在最开始使用root，开启所有controller权限，之后不需要root。那么问题就是安全性如何保证？虽然这里是rootless了，但是权限都拿到了，啥都可以做了。

# nsdelegate

cgroup v2有一个mount option： nsdelegate。[参见文档](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#delegation)。

>### Model of Delegation
A cgroup can be delegated in two ways. First, to a less privileged user by granting write access of the directory and its “cgroup.procs”, “cgroup.threads” and “cgroup.subtree_control” files to the user. Second, if the “nsdelegate” mount option is set, automatically to a cgroup namespace on namespace creation.
Because the resource control interface files in a given directory control the distribution of the parent’s resources, the delegatee shouldn’t be allowed to write to them. For the first method, this is achieved by not granting access to these files. For the second, the kernel rejects writes to all files other than “cgroup.procs” and “cgroup.subtree_control” on a namespace root from inside the namespace.
The end results are equivalent for both delegation types. Once delegated, the user can build sub-hierarchy under the directory, organize processes inside it as it sees fit and further distribute the resources it received from the parent. The limits and other settings of all resource controllers are hierarchical and regardless of what happens in the delegated sub-hierarchy, nothing can escape the resource restrictions imposed by the parent.
Currently, cgroup doesn’t impose any restrictions on the number of cgroups in or nesting depth of a delegated sub-hierarchy; however, this may be limited explicitly in the future.

> ### Delegation Containment
A delegated sub-hierarchy is contained in the sense that processes can’t be moved into or out of the sub-hierarchy by the delegatee.
For delegations to a less privileged user, this is achieved by requiring the following conditions for a process with a non-root euid to migrate a target process into a cgroup by writing its PID to the “cgroup.procs” file.
The writer must have write access to the “cgroup.procs” file.
The writer must have write access to the “cgroup.procs” file of the common ancestor of the source and destination cgroups.
The above two constraints ensure that while a delegatee may migrate processes around freely in the delegated sub-hierarchy it can’t pull in from or push out to outside the sub-hierarchy.
For an example, let’s assume cgroups C0 and C1 have been delegated to user U0 who created C00, C01 under C0 and C10 under C1 as follows and all processes under C0 and C1 belong to U0:
Let’s also say U0 wants to write the PID of a process which is currently in C10 into “C00/cgroup.procs”. U0 has write access to the file; however, the common ancestor of the source cgroup C10 and the destination cgroup C00 is above the points of delegation and U0 would not have write access to its “cgroup.procs” files and thus the write will be denied with -EACCES.
For delegations to namespaces, containment is achieved by requiring that both the source and destination cgroups are reachable from the namespace of the process which is attempting the migration. If either is not reachable, the migration is rejected with -ENOENT.

我在我的fedora32上看，默认的cgroup已经是nsdelegate了，我通过修改systemd配置文件的方式开启了所有controller权限。然后按照文档做了一个试验

```shell
$ mkdir -p c0/c01
$ mkdir -p c1/c11
$ sleep 60 &
[1] 44183
$ echo 44183 > c0/c01/cgroup.procs
$ echo 44183 > c1/c11/cgroup.procs
```

并没有报错...“我”在user.slice下面就是无敌的。

随后我又在网上找到了systemd为user.slice初始化的配置文件：/lib/systemd/system/user@.service

```shell
$ cat /lib/systemd/system/user@.service
#  SPDX-License-Identifier: LGPL-2.1+
#
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=User Manager for UID %i
Documentation=man:user@.service(5)
After=systemd-user-sessions.service user-runtime-dir@%i.service dbus.service
Requires=user-runtime-dir@%i.service
IgnoreOnIsolate=yes

[Service]
User=%i
PAMName=systemd-user
Type=notify
ExecStart=/usr/lib/systemd/systemd --user
Slice=user-%i.slice
KillMode=mixed
Delegate=pids memory
TasksMax=infinity
TimeoutStopSec=120s
KeyringMode=inherit
```

user@.service是为所有的user-uid创建的，所以必须要用sudo reload。

但是有没有办法为user-1000设置delegation呢？这样就不用root了。我在本地没试出来，但是看到[github](https://github.com/containers/podman/issues/1429#issuecomment-419913822)有一段摘自cgroup v2 mail list的一段话，貌似是可能的：

> Or in other words: if you are looking for a way to get your own
per-user delegated cgroup subtree, simply ask systemd for it by
setting Delegate=yes in your service unit file, or by asking for
a scope unit to be registered, also with Delegate=yes set. Nothing
else is supported.


Anyway，文档里面好多陈述有矛盾，有可能是文档比较落后吧。有待研究。

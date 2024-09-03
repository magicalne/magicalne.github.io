---
title: 如何在user namespace中mount proc
layout: post
date: 2020-11-22
tag:
- linux
- user namespace
---

最近在用rust实现一个[oci runtime](https://github.com/opencontainers/runtime-spec)。开始就遇到了很多问题，从这里开始理解到底什么是system programming。

那么什么是system programming？

参考[Kamal Marhubi](http://kamalmarhubi.com/blog/2016/04/13/rust-nix-easier-unix-systems-programming-3/)的答案：

>Systems programming is programming where you spend more time reading man pages than reading the internet.

最近真是读了好多man。。。因为自己想要的东西是google搜不出来的。

我一开始的想法很简单，创建一个user namespace，设置相应的flags，mount devices和fs，然后把pid放到cgroup v2下面，打完收功。没想到第一个步就走不通。

这里参考的user namespace的[example](https://www.man7.org/linux/man-pages/man7/user_namespaces.7.html#EXAMPLES)。然后开始用rust改这里的逻辑。

创建一个新的user namespace流程：

- unshare + all flags
- update uid_map|gid_map
- mount proc
- mount / MS_PRIVATE
- mount rootfs
- mkdir rootfs/oldrootfs
- pivot_root
- chdir /
- umount2 oldrootfs
- rmdir oldrootfs

## 遇到的第一个问题是不能修改gid_map

这个问题很简单：

>Linux 3.19 made a change in the handling of setgroups(2) and the
'gid_map' file to address a security issue. The issue allowed
*unprivileged* users to employ user namespaces in order to drop
The upshot of the 3.19 changes is that in order to update the
'gid_maps' file, use of the setgroups() system call in this
user namespace must first be disabled by writing "deny" to one of
the /proc/PID/setgroups files for this namespace.  That is the
purpose of the following function.

在修改gid_map之前setgroups deny就好。

## 第二个问题是不能直接mount proc

使用命令行很容易复现：

```shell
$unshare -r --pid --mount-proc readlink /proc/self bash 
#unshare: mount /proc failed: Operation not permitted
```

但是使用fork是可以的：

```shell
$unshare -r --fork --pid --mount-proc readlink /proc/self bash
#1
```

这里可以看到，在创建的namespace中，pid是从1开始的。

参考man：

>   -f, --fork
              Fork the specified program as a child process of unshare rather than running it directly.  This is useful when creating a new PID namespace.

最初我在userns_exec_child.c中，尝试在执行execv之前，加了一段mount：

```c
if (mount("proc", "/proc", "proc", MS_REC|MS_BIND, NULL) < 0) {
  fprintf(stderr, "ERROR: %s\n",
      strerror(errno));
  exit(EXIT_FAILURE);
```

结果是一样的，也是没有权限。在google搜索一番之后，找到了[redhat bugzilla](https://bugzilla.redhat.com/show_bug.cgi?id=1390057)。一看这时间，了不起阿，2016-10-31。

Karel Zak解释到：

>IMHO require CLONE_NEWPID to unshare also the /proc makes sense.
And I guess that CLONE_NEWPID requires fork() to create a "init" process in the new pid namespace -- without fork() the new /proc will not contain any process.
\
\
$ unshare --user --pid --fork --mount-proc
\
\
seems correct.

user namespace在mount proc这里的实现是依赖fork的，因为在mount proc之前/proc中只包含当前的pid，如何将当前的pid`修改`成1？显然这是一个immutable的实现，pid是不可以直接修改的。必须依赖fork创建新pid，通过这种方式init。

另外一个信息：

>The --mount-proc has been designed for CLONE_NEWPID


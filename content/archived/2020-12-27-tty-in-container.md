---
title: tty、pty、ptmx、pts与container
layout: post
date: 2020-12-27
tag:
- ptmx
---

当我在实现linux container的时候，遇到的最大的问题（至今为止），就是如何让host与container通过命令行通信。所以深入学习了linux的tty。

所谓tty就是最早的tele-type输入设备，后来也叫pty。

现在的问题来了。我在linux上创建了一个container，这个container有自己的namespace，且会通过fork创建另外一个进程。这里把host进程称为pm，container进程称为ps。那么pm和ps之间如何在terminal中通信呢？pm是我们在terminal里启动的进程，可以通过stdio`看到`pm现在发生着什么。问题是ps是另外一个进程阿，ps是死是活，我怎么通过当前的terminal感知到呢？

所以需要重定向stdio，把ps的stderr、stdout都发送到pm的stdin，并把pm的stdio发送到ps的stdin。

以host的视角看container的目录结构：

```sh
# ls my_container/rootfs       
bin  dev  etc  home  notify  proc  root  sys  tmp  usr  var
```

但是以container的视角看自己当然是这样的：

```sh
# ls /       
bin  dev  etc  home  notify  proc  root  sys  tmp  usr  var
```

# [devpts](https://www.kernel.org/doc/html/latest/filesystems/devpts.html)

devpts历史众多，这里记录了我关注的部分。

在比较new的早期，devpts在linux下是一个单例，如果需要创建一个`新的`terminal，可以通过创建一个pts来实现，而devpts这个设备仍然是独一份。但是时间来到2008年左右，由于linux加入了container的概念，事情就很不一样了。如果devpts还是单例，那么会有安全问题：container1创建了一个pts1，container2可以通过devpts`看到`pts1。为了解决这个问题，devpts增加了`newinstance`这个参数，至此，container中的devpts才是单例。为了可以向后兼容（这个兼容我没看出有啥用），还需要:
```sh
 mount -o bind /dev/pts/ptmx /dev/ptmx
 ```

[patch详见这里](https://lwn.net/Articles/689539/)。

重点在这里：

>4. If multi-instance mode mount is needed for containers, but the system
   startup scripts have not yet been updated, container-startup scripts
   should bind mount /dev/ptmx to /dev/pts/ptmx to avoid breaking single-
   instance mounts.
\
   Or, in general, container-startup scripts should use:\
\
	mount -t devpts -o newinstance -o ptmxmode=0666 devpts /dev/pts\
	if [ ! -L /dev/ptmx ]; then\
		mount -o bind /dev/pts/ptmx /dev/ptmx\
	fi\
\
   When all devpts mounts are multi-instance, /dev/ptmx can permanently be
   a symlink to pts/ptmx and the bind mount can be ignored.

从这里也就能够解释为什么会分别存在/dev/ptmx和/dev/pts/ptmx。在container中，需要通过上述方式为container创建一个新的ptmx。ptmx、pts是成对出现的 - [pseudoterminal master and slave](https://linux.die.net/man/4/ptmx)。

## 为container设置terminal

在初始化namespaces、unshare之后，在fock的子进程中首先需要执行[setsid](https://man7.org/linux/man-pages/man2/setsid.2.html)。这一步是为了确保子进程是`the leader of the new session`，否则在后面执行ioct的时候，会出现eperm。

接下来是按照config.json为container初始化fs、mount ptmx、按照[oci linux的标准](https://github.com/opencontainers/runtime-spec/blob/master/runtime-linux.md)，还需要为container进程映射stdio。然后就可以设置terminal了。rust实现：

```rust
fn setup_terminal() -> Result<RawFd> {
    let master: RawFd = open(
        Path::new("/dev/ptmx"),
        OFlag::O_RDWR | OFlag::O_NOCTTY | OFlag::O_CLOEXEC,
        Mode::empty(),
    )
    .expect("open");
    let slave_name = ptsname_r(master).expect("Failed to create pty slave");
    dbg!(&slave_name);
    grantpt(master).expect("cannot grant");
    unlockpt(master).expect("Failed to unlock");
    let slave_fd: RawFd = open(Path::new(&slave_name), OFlag::O_RDWR, Mode::empty())
        .expect("Failed to open slave fd");
    let console = Path::new("/dev/console");
    if !console.exists() {
        File::create(console).expect("cannot create console");
    }
    mount::<_, _, str, Path>(
        Some(Path::new(&slave_name)),
        console,
        Some("bind"),
        MsFlags::MS_BIND,
        None,
    )?;
    dup3(slave_fd, 0, OFlag::O_RDONLY).expect("Failed to set stdin");
    dup3(slave_fd, 1, OFlag::O_RDONLY).expect("Failed to set stdout");
    dup3(slave_fd, 2, OFlag::O_RDONLY).expect("Failed to set stderr");
    ioctl(0)?; //libc::ioctl(fd, TIOCSCTTY, 0);
    Ok(master)
}
```

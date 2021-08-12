---
title: Checking more file descriptors 
categories: [Howtos]
tags: linux
---

In the previous post we looked at how to check network connections for a
particular process from the `/proc` filesystem. Now let's take a look at the
other file descriptors and see what information we can get from them.

I will be using the same environment as before and I am going to run all
commands inside the Jenkins container. Because the jenkins process has process
id `6`, all information related to this process will be obtained from
`/proc/6/`.

## The list of file descriptors

These are the file descriptors for the Jenkins process. I am removing some of
the lines to keep the output more readable.
```bash
jenkins@54e9a2cefd94:/$ ls -l /proc/6/fd
total 0
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 0 -> /dev/null
l-wx------ 1 jenkins jenkins 64 Aug 10 07:51 1 -> 'pipe:[606872]'
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 10 -> /dev/random
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 100 -> /var/jenkins_home/war/WEB-INF/lib/robust-http-client-1.2.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 101 -> /var/jenkins_home/war/WEB-INF/lib/self-signed-cert-generator-1.0.0.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 102 -> /var/jenkins_home/war/WEB-INF/lib/sezpoz-1.13.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 103 -> /var/jenkins_home/war/WEB-INF/lib/slave-installer-1.7.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 104 -> /var/jenkins_home/war/WEB-INF/lib/slf4j-api-1.7.30.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 105 -> /var/jenkins_home/war/WEB-INF/lib/slf4j-jdk14-1.7.30.jar
...
...
...
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 94 -> /var/jenkins_home/war/WEB-INF/lib/localizer-1.31.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 95 -> /var/jenkins_home/war/WEB-INF/lib/log4j-over-slf4j-1.7.30.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 96 -> /var/jenkins_home/war/WEB-INF/lib/memory-monitor-1.9.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 97 -> /var/jenkins_home/war/WEB-INF/lib/mxparser-1.2.1.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 98 -> /var/jenkins_home/war/WEB-INF/lib/relaxngDatatype-20020414.jar
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 99 -> /var/jenkins_home/war/WEB-INF/lib/remoting-4.7.jar
jenkins@54e9a2cefd94:/$
``` 

## File descriptors pointing to normal files

Most of the file descriptors are pointing to _.jar_ files in
`/var/jenkins_home/war/`. Let's examine one of them:
```
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 90 -> /var/jenkins_home/war/WEB-INF/lib/jzlib-1.1.3-kohsuke-1.jar
```

We can get some details about it from `fdinfo`:
```bash
jenkins@54e9a2cefd94:/$ cat /proc/6/fdinfo/90
pos:	44743
flags:	02100000
mnt_id:	2330
```

We can see the file offset, which is 44743, the flags for access and status, and
the mount id. The flags are displayed in octal and are defined
[here](https://github.com/torvalds/linux/blob/master/include/uapi/asm-generic/fcntl.h).
For this file the flags are `O_RDONLY`, `O_LARGEFILE` and `O_CLOEXEC`.
Information about each flag is available in the [open(2) man
page](https://www.man7.org/linux/man-pages/man2/open.2.html). The mount id shows
the mount point where the file is located. The list of mount points is available
in `mountinfo`:
```bash
jenkins@54e9a2cefd94:/$ grep 2330 /proc/6/mountinfo
2330 2310 252:5 /var/snap/docker/common/var-lib-docker/volumes/bdbbeffbd9fc8b5bc8fac408f38fcf48368d78161070a52fa6d8f284ac525ff2/_data /var/jenkins_home rw,relatime master:1 - ext4 /dev/vda5 rw,errors=remount-ro
jenkins@54e9a2cefd94:/$
```

## Standard input/output and pipes

If we exclude normal files, we have a much smaller set of file descriptors left:
```bash
jenkins@54e9a2cefd94:/$ ls -l /proc/6/fd | grep -v "/var\|/usr\|/opt\|tmp"
total 0
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 0 -> /dev/null
l-wx------ 1 jenkins jenkins 64 Aug 10 07:51 1 -> pipe:[606872]
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 10 -> /dev/random
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 11 -> /dev/urandom
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 12 -> /dev/urandom
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 137 -> socket:[607014]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 138 -> socket:[607015]
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 139 -> pipe:[607020]
l-wx------ 1 jenkins jenkins 64 Aug 10 07:51 140 -> pipe:[607020]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 141 -> anon_inode:[eventpoll]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 147 -> socket:[608390]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 18 -> socket:[606974]
l-wx------ 1 jenkins jenkins 64 Aug 10 07:51 2 -> pipe:[606873]
lr-x------ 1 jenkins jenkins 64 Aug 10 07:13 62 -> pipe:[606950]
lr-x------ 1 jenkins jenkins 64 Aug 10 07:13 63 -> pipe:[606946]
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 7 -> /dev/random
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 8 -> /dev/urandom
lr-x------ 1 jenkins jenkins 64 Aug 10 07:51 9 -> /dev/random
jenkins@54e9a2cefd94:/$
```

File descriptor `0`, `1` and `2` are the `stdin`, `stdout` and `stderr`. `stdin`
is linked to `/dev/null`, so the input is not used. `stdout` and `stderr` are
each redirected to a pipe. Each pipe has an inode, which we can use to find
which process is reading from them:
```bash
jenkins@54e9a2cefd94:/$ ls -l /proc/*/fd/* 2>/dev/null | grep '606872'
l-wx------ 1 jenkins jenkins 64 Aug 10 07:13 /proc/1/fd/1 -> pipe:[606872]
l-wx------ 1 jenkins jenkins 64 Aug 10 07:51 /proc/6/fd/1 -> pipe:[606872]
jenkins@54e9a2cefd94:/$
```

This pipe seems to be used by Jenkins (PID 6) and the launcher process (PID 1),
but they both have file descriptor `1`(`stdout`) pointing to it. Very likely the
pipe is read outside the container. If we check on the host, we see that
`containerd-shim` is reading
from this pipe:
```bash
jenkins@54e9a2cefd94:/$ exit
root@leo-ubuntu-20:~# ls -l /proc/*/fd/* 2>/dev/null | grep '606872'
lr-x------ 1 root             root             64 aug 10 10:32 /proc/26845/fd/11 -> pipe:[606872]
l-wx------ 1 leo              leo              64 aug 10 10:13 /proc/26863/fd/1 -> pipe:[606872]
l-wx------ 1 leo              leo              64 aug 10 10:28 /proc/26890/fd/1 -> pipe:[606872]
root@leo-ubuntu-20:~# ps 26845
    PID TTY      STAT   TIME COMMAND
  26845 ?        Sl     0:04 containerd-shim -namespace moby -workdir /var/snap/docker/common/var-lib-d
root@leo-ubuntu-20:~#
```
The other two processes listed above, with file descriptor `1`(`stdout`)
pointing to the pipe are the jenkins process and the launcher process as seen by
the host.

## Other file descriptors

In the file descriptor list we see `/dev/random` and `/dev/urandom`. I don't
know why these would need to be opened more than once.

Another file descriptor in the list is linked to `anon_inode:[eventpoll]`. This
is used for monitoring multiple file descriptor and getting notification when an
I/O operation is possible.

All other file descriptors are for sockets. Network sockets in the
previous post, and unix sockets do not have information easily available to
identify the other end.

 
## References

For more details about the `/proc/` filesystem and the other details presented
here, please check:
- [proc man page](https://man7.org/linux/man-pages/man5/proc.5.html)
- [/proc filesystem documentation](https://www.kernel.org/doc/html/latest/filesystems/proc.html)
- [file open flags](https://github.com/torvalds/linux/blob/master/include/uapi/asm-generic/fcntl.h)
- [open(2) man
page](https://www.man7.org/linux/man-pages/man2/open.2.html)
- [epoll(7) man page](https://man7.org/linux/man-pages/man7/epoll.7.html)
- [random(4) man page](https://man7.org/linux/man-pages/man4/random.4.html)

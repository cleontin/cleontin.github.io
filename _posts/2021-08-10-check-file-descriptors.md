---
title: How to get process socket information from /proc 
categories: [Howtos]
tags: linux
---

When I want to see the connections opened by some process I normally use
commands like `lsof` and `ss`. But these commands are not always available.
In these scenarios, I found that it's not that difficult to obtain the same
information from the `/proc` filesystem.

I will show here how this can be done by first checking the output from `lsof`
and `ss`, then finding the same information from the `proc` filesystem.

## Test setup

For this test, I will start a Jenkins container. The container will have TCP
port 8080 open, and I will access it from my browser to generate connections.
The socket inodes and the port numbers will change during the tests, because the
connections are not kept open for more than a few seconds.

We will be looking at the connections as seen from the host and from inside the
container. We will see that network information fro inside the container is not
directly available to commands running on the host. Also we will see that `lsof`
is not available inside the container. Finally we will examine connection
information by looking in the `/proc` filesystem inside the container.


## Connections as seen by the host

First let's find the id of the process as seen by the host:
```bash
root@leo-ubuntu-20:~# ps awwwxf | grep jenkins
  26863 ?        Ss     0:00  |       \_ /sbin/tini -- /usr/local/bin/jenkins.sh
  26890 ?        Sl     0:56  |           \_ java -Duser.home=/var/jenkins_home -Djenkins.model.Jenkins.slaveAgentPort=50000 -jar /usr/share/jenkins/jenkins.war
  27312 pts/2    S+     0:00                          \_ grep --color=auto jenkins
root@leo-ubuntu-20:~#
```

We see that the PID is `26890`.
Now let's check the open files as listed by lsof. I'm filtering by `sock` to
avoid the huge list of open files:
```bash
root@leo-ubuntu-20:~# lsof -p 26890 | grep sock
lsof: WARNING: can't stat() fuse.gvfsd-fuse file system /run/user/1000/gvfs
      Output information may be incomplete.
lsof: WARNING: can't stat() fuse file system /run/user/1000/doc
      Output information may be incomplete.
java    26890  leo   18u     sock    0,8      0t0 606974 protocol: UNIX
java    26890  leo  137u     sock    0,8      0t0 607014 protocol: TCP
java    26890  leo  138u     sock    0,8      0t0 607015 protocol: UNIX
java    26890  leo  142u     sock    0,8      0t0 618040 protocol: TCP
java    26890  leo  145u     sock    0,8      0t0 618049 protocol: TCP
java    26890  leo  146u     sock    0,8      0t0 617834 protocol: TCP
java    26890  leo  147u     sock    0,8      0t0 608390 protocol: TCP
java    26890  leo  149u     sock    0,8      0t0 618055 protocol: TCP
java    26890  leo  150u     sock    0,8      0t0 618061 protocol: TCP
java    26890  leo  152u     sock    0,8      0t0 617295 protocol: TCP
root@leo-ubuntu-20:~#
```

We can see two UNIX sockets and a few TCP sockets. The TCP information is  not
available, as Jenkins is running in a different network namespace from the host.

If we try using `ss` to see the open connections, we hit the same limitation.
The information is not available in the network namespace of the host:
```bash
root@leo-ubuntu-20:~# ss -p | grep 26890
root@leo-ubuntu-20:~#
```

But if we run the same command in the namespace of the process we can see
detailed information about the connections:
```bash
root@leo-ubuntu-20:~# nsenter -t 26890 -n ss -p | grep 26890
u_str  ESTAB  0       0                      * 607015                  * 0       users:(("java",pid=26890,fd=138))
u_str  ESTAB  0       0                      * 606974                  * 0       users:(("java",pid=26890,fd=18))
tcp    ESTAB  0       0             172.17.0.2:http-alt       172.17.0.1:54584   users:(("java",pid=26890,fd=149))
tcp    ESTAB  0       0             172.17.0.2:http-alt       172.17.0.1:54590   users:(("java",pid=26890,fd=151))
tcp    ESTAB  0       0             172.17.0.2:http-alt       172.17.0.1:54586   users:(("java",pid=26890,fd=150))
tcp    ESTAB  0       0             172.17.0.2:http-alt       172.17.0.1:54580   users:(("java",pid=26890,fd=142))
tcp    ESTAB  0       0             172.17.0.2:http-alt       172.17.0.1:54570   users:(("java",pid=26890,fd=153))
tcp    ESTAB  0       0             172.17.0.2:http-alt       172.17.0.1:54582   users:(("java",pid=26890,fd=146))
root@leo-ubuntu-20:~#
```

## Connections as seen inside the container

We now enter inside the container. We will run all further commands from here:
```bash
root@leo-ubuntu-20:~# docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                               NAMES
54e9a2cefd94        jenkins/jenkins:lts   "/sbin/tini -- /usr/…"   34 minutes ago      Up 34 minutes       0.0.0.0:8080->8080/tcp, 50000/tcp   thirsty_albattani
root@leo-ubuntu-20:~# docker exec -ti 54e9 bash
jenkins@54e9a2cefd94:/$
```

Let's check the process IDs here:
```bash
jenkins@54e9a2cefd94:/$ ps awwxf
    PID TTY      STAT   TIME COMMAND
     92 pts/0    Ss     0:00 bash
     97 pts/0    R+     0:00  \_ ps awwxf
      1 ?        Ss     0:00 /sbin/tini -- /usr/local/bin/jenkins.sh
      6 ?        Sl     1:00 java -Duser.home=/var/jenkins_home -Djenkins.model.Jenkins.slaveAgentPort=50000 -jar /usr/share/jenkins/jenkins.war
jenkins@54e9a2cefd94:/$
```

And let's see what information we can collect with `lsof` and `ss`:
```bash
jenkins@54e9a2cefd94:/$ lsof -p 6
bash: lsof: command not found
jenkins@54e9a2cefd94:/$ ss -p
Netid State Recv-Q Send-Q   Local Address:Port       Peer Address:Port
u_str ESTAB 0      0                    * 607015                * 0      users:(("java",pid=6,fd=138))
u_str ESTAB 0      0                    * 606974                * 0      users:(("java",pid=6,fd=18))
tcp   ESTAB 0      0           172.17.0.2:http-alt     172.17.0.1:54730  users:(("java",pid=6,fd=146))
tcp   ESTAB 0      0           172.17.0.2:http-alt     172.17.0.1:54718  users:(("java",pid=6,fd=152))
tcp   ESTAB 0      0           172.17.0.2:http-alt     172.17.0.1:54722  users:(("java",pid=6,fd=142))
tcp   ESTAB 0      0           172.17.0.2:http-alt     172.17.0.1:54728  users:(("java",pid=6,fd=145))
jenkins@54e9a2cefd94:/$
```

We can see that `lsof` is not available and `ss` provides the full information.
Now let's take a look at the  information from `/proc`.

## Checking the /proc filesystem

In the `/proc` filesystem there is a folder for each process id. Each of these
folders contain information about that specific process. We will be looking at
the `fd` subfolder, which lists the file descriptors of that process. Again,
I'm filtering on `sock` to avoid the huge list of files:
```bash
jenkins@54e9a2cefd94:/$ ls -l /proc/6/fd | grep sock
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 137 -> socket:[607014]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 138 -> socket:[607015]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 142 -> socket:[619101]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 145 -> socket:[619107]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 146 -> socket:[619108]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 147 -> socket:[608390]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:56 149 -> socket:[618922]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:56 150 -> socket:[619110]
lrwx------ 1 jenkins jenkins 64 Aug 10 07:51 18 -> socket:[606974]
jenkins@54e9a2cefd94:/$
```

We now have the list of sockets but we need to match these to network addresses
and ports. This information can be found in files from `/proc/net/`, for example
`proc/net/tcp` for tcp connections.

Let's take a socket inode from the output above and look for it in all files
from `/proc/net` while discarding any error messages:
```bash
jenkins@54e9a2cefd94:/$ grep 606974 /proc/net/* 2>/dev/null
/proc/net/unix:0000000000000000: 00000002 00000000 00000000 0001 03 606974
jenkins@54e9a2cefd94:/$
```

We can see that this is a unix socket. Now let's  get the same info for all the
sockets.

I will use a longer command to check the file descriptors for process with id
`6`, select the lines matching `socket`, then selecting the text that comes
after `[` and before `]`. For each inode obtained in this way, I will grep
the files in `/proc/net` looking for that specific inode.
```bash
jenkins@54e9a2cefd94:/$ for s in `ls -l /proc/6/fd | grep sock | cut -d'[' -f2 | cut -d ']' -f1` ; do grep $s /proc/net/* 2>/dev/null ; done
/proc/net/tcp:   1: 00000000:1F90 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1000        0 607014 1 0000000000000000 100 0 0 10 0
/proc/net/unix:0000000000000000: 00000002 00000000 00000000 0001 03 607015
/proc/net/tcp:   8: 020011AC:1F90 010011AC:D63E 01 00000000:00000000 00:00000000 00000000  1000        0 621793 1 0000000000000000 20 4 20 10 -1
/proc/net/tcp:   4: 020011AC:1F90 010011AC:D63A 01 00000000:00000000 00:00000000 00000000  1000        0 621699 1 0000000000000000 20 4 3 10 -1
/proc/net/tcp:   6: 020011AC:1F90 010011AC:D642 01 00000000:00000000 00:00000000 00000000  1000        0 621807 1 0000000000000000 20 4 21 10 -1
/proc/net/tcp:   0: 00000000:C350 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1000        0 608390 1 0000000000000000 100 0 0 10 0
/proc/net/tcp:   3: 020011AC:1F90 010011AC:D646 01 00000000:00000000 00:00000000 00000000  1000        0 621808 1 0000000000000000 20 4 19 10 -1
/proc/net/tcp:   5: 020011AC:1F90 010011AC:D64A 01 00000000:00000000 00:00000000 00000000  1000        0 621809 1 0000000000000000 20 4 21 10 -1
/proc/net/tcp:   9: 020011AC:1F90 010011AC:D64E 01 00000000:00000000 00:00000000 00000000  1000        0 621810 1 0000000000000000 20 4 20 10 -1
/proc/net/unix:0000000000000000: 00000002 00000000 00000000 0001 03 606974
jenkins@54e9a2cefd94:/$
```

The information may seem difficult to read because it's written in hex, but it's
just what we needed. Let's check one of the TCP connections:
```
/proc/net/tcp:   8: 020011AC:1F90 010011AC:D63E 01 00000000:00000000 00:00000000 00000000  1000        0 621793 1 0000000000000000 20 4 20 10 -1
```

We can see two fields that look like IP and port information: `020011AC:1F90`
where `02 00 11 AC` is 2, 0, 17, 172 so the IP address is `172.17.0.2`. Port 
`1F90` is 8080 in decimal, so the information here is `172.17.0.2:8080` which is
the local address.
By converting the second field `010011AC:D63E` we get `172.17.0.1:54846` which
is the other end of this connection.

Now we know that this line shows a connection between `172.17.0.2:8080` and
`172.17.0.1:54846`.

There is another line in the output that looks interesting:
```
/proc/net/tcp:   1: 00000000:1F90 00000000:0000 0A 00000000:00000000 00:00000000 00000000  1000        0 607014 1 0000000000000000 100 0 0 10 0
```

Here, the connection looks as if it were from `0.0.0.0:0` to `0.0.0.0:8080`.
This is showing that the process is listening on all local IPs (`0.0.0.0`) on
port `8080`.

So, by looking only at the `/proc` filesystem we found the same information that
was provided by `lsof` and `ss`. This can be useful when these tools are not
available, or maybe even just for fun.

## References

For more details about the `/proc/` filesystem and the other commands used here,
please check:
- [lsof man page](https://man7.org/linux/man-pages/man8/lsof.8.html)
- [ss man page](https://www.man7.org/linux/man-pages/man8/ss.8.html)
- [nsenter man page](https://man7.org/linux/man-pages/man1/nsenter.1.html)
- [proc man page](https://man7.org/linux/man-pages/man5/proc.5.html)
- [/proc filesystem documentation](https://www.kernel.org/doc/html/latest/filesystems/proc.html)

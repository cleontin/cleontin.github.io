---
title: Checking the docker bridge networks
categories: [Howtos]
tags: port
---

This is just for fun and practice. Let's take a look at how docker implements
the bridge networks.

## The environment

I am using the same VM as before, running Ubuntu. I have docker installed with
one container already running (jenkins, used for previous posts). To make it
more interesting, I'm starting a second container running busybox.

I will only look at standard linux commands on the host and inside the
containers.

## Host interfaces

Let's see the interfaces on the host:
```bash
root@leo-ubuntu-20:~# ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:d5:b6:2e brd ff:ff:ff:ff:ff:ff
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:1b:5e:ac:d5 brd ff:ff:ff:ff:ff:ff
5: veth1673542@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether 0e:2b:c3:97:90:36 brd ff:ff:ff:ff:ff:ff link-netnsid 0
11: veth75e137d@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
    link/ether aa:a1:dd:d8:99:0c brd ff:ff:ff:ff:ff:ff link-netnsid 1
root@leo-ubuntu-20:~#
```

I see my loopback interface `lo`, my (almost) physical interface `enp1s0`, a
`docker0` interface and two other interfaces `veth1673542` and `veth60e8510`,
probably used by the containers.

## Interface types

Let's check what type of interfaces they are by looking at what drivers they
use:

```bash
root@leo-ubuntu-20:~# ethtool -i docker0 | grep driver
driver: bridge
root@leo-ubuntu-20:~# ethtool -i veth1673542 | grep driver
driver: veth
root@leo-ubuntu-20:~#
```

Ok. We found that `docker0` is a bridge interface and the `veth` interfaces
are, as expected, virtual ethernet interfaces.

## Brigde inteface

We can get more details about the bridge with the `brctl` command:
```bash
root@leo-ubuntu-20:~# brctl show docker0
bridge name	bridge id		STP enabled	interfaces
docker0		8000.02421b5eacd5	no		veth1673542
							veth75e137d
root@leo-ubuntu-20:~#
```

We can also check what mac addresses were learned:
```bash
root@leo-ubuntu-20:~# brctl showmacs docker0
port no	mac addr		is local?	ageing timer
  1	02:42:ac:11:00:02	no		 209.62
  1	0e:2b:c3:97:90:36	yes		   0.00
  1	0e:2b:c3:97:90:36	yes		   0.00
  2	aa:a1:dd:d8:99:0c	yes		   0.00
  2	aa:a1:dd:d8:99:0c	yes		   0.00
root@leo-ubuntu-20:~#
```

There are three mac addresses known. Two of them are local (I have no idea why
they are listed two times), and one is not local. The local ones are for the
veth interface on the host. The other one is for the interface inside the
container.

The `brctl` command is showing the port number associated with each mac address,
but it took me some time to find what interface corresponds to each port number.
Luckily I found that this information is shown when checking the stp info for
the bridge:
```bash
root@leo-ubuntu-20:~# brctl showstp docker0 | grep veth
veth1673542 (1)
veth75e137d (2)
root@leo-ubuntu-20:~#
``` 

We can see the bridge has the two veth interfaces attached.

## Veth interfaces

The veth interfaces are always created in pairs. Here, information about the
veth pair is part of the naming (`@if4`), but it can also be obtained from
`ethtool`:

```bash
root@leo-ubuntu-20:~# ethtool -S veth1673542
NIC statistics:
     peer_ifindex: 4
     rx_queue_0_xdp_packets: 0
     rx_queue_0_xdp_bytes: 0
     rx_queue_0_drops: 0
     rx_queue_0_xdp_redirect: 0
     rx_queue_0_xdp_drops: 0
     rx_queue_0_xdp_tx: 0
     rx_queue_0_xdp_tx_errors: 0
     tx_queue_0_xdp_xmit: 0
     tx_queue_0_xdp_xmit_errors: 0
root@leo-ubuntu-20:~#
```

We can see `peer_ifindex: 4`, so we are now sure it connects to interface with
index 4. As interface with index 4 was not seen on the host, we can assume it
must be inside one of the containers.


Now let's try to check inside the containers. I'll try the _older_ jenkins
container first:
```bash
root@leo-ubuntu-20:~# docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                               NAMES
f56d0d166082        busybox               "sleep 3600"             2 hours ago         Up 44 minutes                                           goofy_aryabhata
54e9a2cefd94        jenkins/jenkins:lts   "/sbin/tini -- /usr/…"   9 days ago          Up 9 days           0.0.0.0:8080->8080/tcp, 50000/tcp   thirsty_albattani
root@leo-ubuntu-20:~# docker exec 54e9a2cefd94 ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@leo-ubuntu-20:~#
```

It seems like I was lucky. Interface with index 4 is used by the jenkins
container. And the name confirms that it's attached to interface with index 5
(our `veth1673542`). Now let's check with the ethtool command:
```bash
root@leo-ubuntu-20:~# docker exec 54e9a2cefd94 ethtool -S eth0
OCI runtime exec failed: exec failed: container_linux.go:349: starting container process caused "exec: \"ethtool\": executable file not found in $PATH": unknown
root@leo-ubuntu-20:~#
```

It looks like the command is not available inside the container. Let's run it
from the host, but on the network namespace of the jenkins container:
```bash
root@leo-ubuntu-20:~# lsns -t net
        NS TYPE NPROCS    PID USER     NETNSID NSFS COMMAND
4026531992 net     247      1 root  unassigned      /lib/systemd/systemd --system --deserialize 18
4026532415 net       1    558 root  unassigned      /usr/sbin/haveged --Foreground --verbose=1 -w 1024
4026532540 net       1    937 rtkit unassigned      /usr/libexec/rtkit-daemon
4026532622 net       3  26863 leo            0      /sbin/tini -- /usr/local/bin/jenkins.sh
4026532820 net       1 127051 root           1      sleep 3600
root@leo-ubuntu-20:~# nsenter -t 26863 -n ethtool -S eth0
NIC statistics:
     peer_ifindex: 5
     rx_queue_0_xdp_packets: 0
     rx_queue_0_xdp_bytes: 0
     rx_queue_0_drops: 0
     rx_queue_0_xdp_redirect: 0
     rx_queue_0_xdp_drops: 0
     rx_queue_0_xdp_tx: 0
     rx_queue_0_xdp_tx_errors: 0
     tx_queue_0_xdp_xmit: 0
     tx_queue_0_xdp_xmit_errors: 0
root@leo-ubuntu-20:~#
```

We have confirmation that `eth0` interface inside the jenkins container is
connected to interface with index 5 on the host.




## References

Here you can find more details about the commands used and the docker bridge networks:
- [ethtool man page](https://man7.org/linux/man-pages/man8/ethtool.8.html)
- [brctl man page](https://www.man7.org/linux/man-pages/man8/brctl.8.html)
- [veth man page](https://man7.org/linux/man-pages/man4/veth.4.html)
- [lsns man page](https://man7.org/linux/man-pages/man8/lsns.8.html)
- [Docker documentation for bridge
networks](https://docs.docker.com/network/bridge/)

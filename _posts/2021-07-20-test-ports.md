---
title: How to test if a port is open and accessible
categories: [Howtos]
tags: test port
---

Many times while troubleshooting, we need to test if the network connection to a
specific service is working or not. The result from such a test can show if I'm
dealing with a problem on the network side or if there's something wrong with
the service.

This can be tested for both TCP and UDP ports, but since UDP doesn't have the
mechanism for reliable delivery, we cannot easily differentiate between open UDP
ports and filtered ones.

## Possible behaviors

When testing, we can see three possible behaviors:
3. The connection is established
    - this will only be seen with TCP connections that are successful
    - it shows that the port is reachable
    - for UDP we can confirm successful connection only by exchanging messages
with the tested service

2. The connection is not immediately rejected, but also we are not seeing any
replies
    - this shows that either the requests sent are not reaching the port or that
it's replies are not reaching back to our test process
    - this usually means that some firewall is blocking our packets
    - TCP will wait for some time, then will throw some timeout error
    - for UDP ports no reply is expected, so this could also mean that the
connection was successful  - we need to send/receive data through this
connection to see if it really is successful

1. The connection is rejected
    - this usually happens without any delay
    - it usually means that the server is refusing the connection (service is
not running or the firewall is rejecting the traffic)
    - it can also be seen if network firewalls are rejecting the traffic
although this is very unlikely

## Test cases

I the examples below, we will test three TCP and three UDP  ports, showing each
possible behavior:
- TCP port _22_ on ip _127.0.0.1_ (localhost)
    - this port is open and will accept connections
    - will result in behavior #3 from above
- UDP port _53_ on _192.168.3.8_
    - this is a port that will accept connections
    - the behavior is the same as if the port cannot be reached
- TCP port _22_ on _192.168.12.13_
    - this ip address cannot be reached
    - the commands will return a timeout error as described in  #2 from above
- UDP port _53_ on _192.168.12.13_
    - this host does not exist, so the port cannot be reached
    - because UDP does not send confirmations, we cannot differentiate this from
a successful connection
- TCP port _23_ on _127.0.0.1_
    - the port is reachable but has no service listening on it
    - this will result in connection being rejected - behavior #1 from above
- UDP port _54_ on _192.168.3.8_
    - this port is reachable but has no service listening on it
    - we will see the connection being rejected as in #1 above

## Example ouput
### Telnet

The telnet command is a client for the telnet protocol. Because the telnet
protocol is just an unencrypted terminal over a TCP session, we can now use the
telnet command to interact with other unencrypted services (HTTP, SMTP) and to
check connectivity to any TCP port. Telnet cannot connect to UDP ports.
 
Testing the working connection we see the message `Connected to 127.0.0.1.`
showing that the connection was successful.
I am then using `CTRL+]` followed by the q (short for quit) command to close
the session.

```bash
[root@localhost ~]# telnet 127.0.0.1 22
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
SSH-2.0-OpenSSH_8.1
^]

telnet> q
Connection closed.
[root@localhost ~]#
```

If we try with the ip and port that are not reachable, the command will wait for 
the connection for a long time, showing `Trying 192.168.12.13...`.
It will exit after some time with the message `Connection timed out`:
```bash
[root@localhost ~]# telnet 192.168.12.13 22
Trying 192.168.12.13...
telnet: connect to address 192.168.12.13: Connection timed out
[root@localhost ~]#
```

For the connection that is rejected, `Connection refused` will be shown:
```bash
[root@localhost ~]# telnet 127.0.0.1 23
Trying 127.0.0.1...
telnet: connect to address 127.0.0.1: Connection refused
[root@localhost ~]#
```

### Ncat

Ncat (nc command) is a tool used to read and write data from the network.
It has a lot of features and can be used in very complex scenarios.

The `-z` option of the __nc__ command is used for testing a connection.
After the command executes we have to examine the exit code to see if the
connection was successful or not.
To test UDP ports, we must use the `-u` option.

When testing the reachable port, the nc command exits immediately and the exit
code `0` shows a success:
```bash
[root@localhost ~]# nc -z 127.0.0.1 22
[root@localhost ~]# echo $?
0
[root@localhost ~]#
```

Similar output for reachable UDP ports:
```
[root@localhost ~]# nc -zu 192.168.3.8 53
[root@localhost ~]# echo $?
0
[root@localhost ~]#
```

When testing the port that is not reachable, the command takes a few seconds to execute (10s in my case) and the exit code shows there was an error.
```bash
[root@localhost ~]# nc -z 192.168.12.13 23
[root@localhost ~]# echo $?
1
[root@localhost ~]#
```
Because UDP does not send a reply on successful connection, a port that is not 
reachable cannot be distinguished from one that is reachable. For this
unreachable port we see a success code:
```
[root@localhost ~]# nc -zu 192.168.12.13 53
[root@localhost ~]# echo $?
0
[root@localhost ~]#
``` 

If the connection is refused, the command exits immediately and the exit code
also shows an error.
```bash
[root@localhost ~]# nc -z 127.0.0.1 23
[root@localhost ~]# echo $?
1
[root@localhost ~]#
```
For refused connections we can see the error code even for UDP:
```
[root@localhost ~]# nc -zu 192.168.3.8 54
[root@localhost ~]# echo $?
1
[root@localhost ~]#
```

When testing TCP ports with nc, the only difference between a timed out
connection and a refused one is the time it takes for the command to execute.


### Bash redirections

Bash has a special way of handling `/dev/tcp/<ip_address>/<port>` files when
they are used in redirection. Writing to such a file will actually open a TCP
connection to that ip address and port and will send the data over this
connection.

We can use this to test the connection to a specific port by sending some data
to it and checking the result.

For a reachable port, the command exits immediately with a success exit code:
```bash
[root@localhost ~]# echo x > /dev/tcp/127.0.0.1/22
[root@localhost ~]# echo $?
0
[root@localhost ~]#
```
Similar for the UDP port:
```
[root@localhost ~]# echo x > /dev/udp/192.168.3.8/53
[root@localhost ~]# echo $?
0
[root@localhost ~]#
```

When the port is not reachable, the command takes some time to execute and then
exits with a `Connection timed out` message:
```bash
[root@localhost ~]# echo x > /dev/tcp/192.168.12.13/23
-bash: connect: Connection timed out
-bash: /dev/tcp/192.168.12.13/23: Connection timed out
[root@localhost ~]#
```

The unreachable UDP port looks just like the reachable one:
```
[root@localhost ~]# echo x > /dev/udp/192.168.12.13/53
[root@localhost ~]# echo $?
0
[root@localhost ~]#
```

When the connection is refused, it exits immediately with the message:
`Connection refused`:
```bash
[root@localhost ~]# echo x > /dev/tcp/127.0.0.1/23
-bash: connect: Connection refused
-bash: /dev/tcp/127.0.0.1/23: Connection refused
[root@localhost ~]#
```

Strangely, for the refused UDP connection we don't see any error. I don't know how to explain this right now:
```
[root@localhost ~]# echo x > /dev/udp/192.168.3.8/54
[root@localhost ~]# echo $?
0
[root@localhost ~]#
```

It looks like bash redirection is not useful at all in testing UDP ports. I will try to understand this and update this post.

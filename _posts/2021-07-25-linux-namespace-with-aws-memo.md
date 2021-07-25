---
layout: post
title: "Linux Namespace with AWS Memo"
description: ""
category: 
tags: ["aws", "linux", "namespace", "net"]
---

## UTS Namespace

[uts_namespaces(7)](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)

Create a namespace file and mount it by `unshare` command. Note that `sudo` is required here for mounting. It's not for changing the hostname. In "User Namespace" section, a method without `sudo` will be shown.

Then, you can enter the namespace through the mount point by `nsenter`.
```
ubuntu@azrle-main:~$ touch foo.uts
ubuntu@azrle-main:~$ sudo unshare --uts=foo.uts hostname container-test
ubuntu@azrle-main:~$ sudo nsenter --uts=foo.uts uname -n
container-test
```

You can also find mounted namespace filesystem by `findmnt`.
```
ubuntu@azrle-main:~$ findmnt -t nsfs
TARGET                SOURCE                 FSTYPE OPTIONS
/run/snapd/ns/lxd.mnt nsfs[mnt:[4026532200]] nsfs   rw
/home/ubuntu/foo.uts  nsfs[uts:[4026532188]] nsfs   rw
```

To clean up, `umount` the namespace.
```
sudo umount foo.uts
rm foo.uts
```

## User Namespace

[user_namespaces(7)](https://man7.org/linux/man-pages/man7/user_namespaces.7.html)

We can create an user namespace and map users into the namespace.

For example, we can map the current user to the `root` user in the namespace by `--map-root-user` option when we use `unshare`.

```
ubuntu@azrle-main:~$ unshare --uts --map-root-user --user bash -c 'hostname container-test; bash'

## Inside the namespace (container)
root@container-test:~# uname -n
container-test
### We can check uid mapping root(0) <-> ubuntu(1000)
root@container-test:~# cat /proc/self/uid_map
         0       1000          1

### Create a file inside container
root@container-test:~# touch /tmp/from-container
root@container-test:~# ls -l /tmp/from-container
-rw-rw-r-- 1 root root 0 Jul 25 10:58 /tmp/from-container

## Outside the namespace (host)
ubuntu@azrle-main:~$ lsns -t user
        NS TYPE  NPROCS    PID USER   COMMAND
4026531837 user      13   1933 ubuntu /lib/systemd/systemd --user
4026532205 user       2 586301 ubuntu bash -c hostname container-test; bash

ubuntu@azrle-main:~$ ls -l /tmp/from-container
-rw-rw-r-- 1 ubuntu ubuntu 0 Jul 25 10:58 /tmp/from-container
ubuntu@azrle-main:~$ touch /tmp/from-host-ubuntu
ubuntu@azrle-main:~$ sudo -uwww-data touch /tmp/from-host-www

## Inside the namepace (container)
root@container-test:~# ls -l /tmp/from-*
-rw-rw-r-- 1 root   root    0 Jul 25 10:58 /tmp/from-container
-rw-rw-r-- 1 root   root    0 Jul 25 11:00 /tmp/from-host-ubuntu
-rw-rw-r-- 1 nobody nogroup 0 Jul 25 11:01 /tmp/from-host-www

root@container-test:~# stat /tmp/from-host-www
  File: /tmp/from-host-www
  Size: 0         	Blocks: 0          IO Block: 4096   regular empty file
Device: 10301h/66305d	Inode: 2324        Links: 1
Access: (0664/-rw-rw-r--)  Uid: (65534/  nobody)   Gid: (65534/ nogroup)
Access: 2021-07-25 11:01:09.569230115 +0000
Modify: 2021-07-25 11:01:09.569230115 +0000
Change: 2021-07-25 11:01:09.569230115 +0000
 Birth: -
```

Note that Uid for unmapped user and group IDs are `65534`.

## PID namespace

[pid_namespaces(7)](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)

```
ubuntu@azrle-main:~$ unshare --uts --map-root-user --user --pid --mount bash -c 'date; date'
Sun 25 Jul 2021 11:15:38 AM UTC
bash: fork: Cannot allocate memory
```

The second `date` encounters an `ENOMEM` error. The reason behind is that the first `date` is the init process for the namespace and it ends. As the result, no more process can be created inside the namespace as the init is dead. Attempts will fail with `ENOMEM` error.

To solve this, we need `--fork` option.
```
ubuntu@azrle-main:~$ unshare --uts --map-root-user --user --pid --mount --fork bash -c 'date; date'
Sun 25 Jul 2021 11:19:54 AM UTC
Sun 25 Jul 2021 11:19:54 AM UTC
```

Let's check PIDs too.
```
ubuntu@azrle-main:~$ unshare --uts --map-root-user --user --pid --mount --fork bash -c 'hostname container-test; bash'
root@container-test:~# ps
    PID TTY          TIME CMD
 586292 pts/5    00:00:00 bash
 586554 pts/5    00:00:00 unshare
 586555 pts/5    00:00:00 bash
 586557 pts/5    00:00:00 bash
 586563 pts/5    00:00:00 ps
 ```

Looks like they are not isolated from the host.
This is because that `/proc` is not mounted for the PID namespace actually.

After mounting the PID namespace, we can observe PIDs are changed.
```
root@container-test:~# mount -t proc proc /proc/
root@container-test:~# ps
    PID TTY          TIME CMD
      1 pts/5    00:00:00 bash
      3 pts/5    00:00:00 bash
     13 pts/5    00:00:00 ps
```

Note that there is a shortcut for it - `--mount-proc` option.
```
ubuntu@azrle-main:~$ unshare --uts --map-root-user --user --pid --mount --fork --mount-proc bash -c 'hostname container-test; bash'
root@container-test:~# ps auxwww
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.7   7024  3596 pts/5    S    11:26   0:00 bash -c hostname container-test; bash
root           3  0.0  1.0   8264  4920 pts/5    S    11:26   0:00 bash
root          10  0.0  0.7   8892  3344 pts/5    R+   11:27   0:00 ps auxwww
```

## Mount Namespace

[mount_namespaces(7)](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)

We have already seen the mount namespace. Let's see another option to switch root directory.

```
ubuntu@azrle-main:~$ env - unshare --uts --map-root-user --user --pid --mount --fork --mount-proc -R /home/ubuntu/rootfs/alphine sh -c 'hostname container-test; sh'
/ # uname -n | tee /tmp/from-container
container-test

## Outside the namespace (host)
ubuntu@azrle-main:~$ cat /home/ubuntu/rootfs/alphine/tmp/from-container
container-test
```

## Network Namespace

[network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)

### Setup
* foo (Primary IP: 172.31.37.209, Secondary IP: 172.31.37.240)
* bar (Primary IP: 172.31.43.37, Secondary IP: 172.31.37.27)

[AWS UserGuide - Multiple IP]https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/MultipleIP.html

This demo is using secondary private IP on the same network interface (ENI) with the primary private IP. You can also different ENI for the similar purpose.

### Inside VPC
```
ubuntu@foo:~$ unshare --uts --map-root-user --user \
    --pid --mount --fork --mount-proc \
    --net \
    bash -c 'hostname container-foo; bash'

root@container-foo:~# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

As it shows, we only have one loopback interface here. Of course, nowhere can be access from the namespace `container-foo` now.
```
root@container-foo:~# ping -c 3 172.31.37.209  # foo host
ping: connect: Network is unreachable
root@container-foo:~# ping -c 3 172.31.43.37   # bar host
ping: connect: Network is unreachable
```

Now we create paired `veth` interfaces. One end is inside the namespace (container) and the other end is ouside the namespace (host).

First, create paired interfaces `veth_host_foo` and `veth_guest_foo`.
```
# On the host
ubuntu@foo:~$ sudo ip link add veth_host_foo type veth peer name veth_guest_foo
ubuntu@foo:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 0e:db:05:04:45:99 brd ff:ff:ff:ff:ff:ff
    inet 172.31.37.209/20 brd 172.31.47.255 scope global dynamic ens5
       valid_lft 2629sec preferred_lft 2629sec
    inet6 fe80::cdb:5ff:fe04:4599/64 scope link
       valid_lft forever preferred_lft forever
3: veth_guest_foo@veth_host_foo: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:fe:54:6a:f8:fd brd ff:ff:ff:ff:ff:ff
4: veth_host_foo@veth_guest_foo: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 66:29:2b:4d:aa:49 brd ff:ff:ff:ff:ff:ff
```

Then, move `veth_guest_foo` into the namespace.
```
ubuntu@foo:~$ lsns -t net
        NS TYPE NPROCS   PID USER      NETNSID NSFS COMMAND
4026531992 net       4  1588 ubuntu unassigned      /lib/systemd/systemd --user
4026532205 net       3  1701 ubuntu unassigned      unshare --uts --map-root-user --user --pid --mount --fork --moun

ubuntu@foo:~$ sudo ip netns attach net-foo 1701
ubuntu@foo:~$ sudo ip link set veth_guest_foo netns net-foo

# only one end is remaining in the host
ubuntu@foo:~$ ip --br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
ens5             UP             172.31.37.209/20 fe80::cdb:5ff:fe04:4599/64
veth_host_foo@if3 DOWN

# bring the interface up
ubuntu@foo:~$ sudo ip link set dev veth_host_foo up

# also note the routing table
ubuntu@foo:~$ ip --br r
default via 172.31.32.1 dev ens5 proto dhcp src 172.31.37.209 metric 100
172.31.32.0/20 dev ens5 proto kernel scope link src 172.31.37.209
172.31.32.1 dev ens5 proto dhcp scope link src 172.31.37.209 metric 100
```

Now we can check the network interface inside the namespace (container) and bring it up.
```
# Inside the namespace

root@container-foo:~# ip a
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth_guest_foo@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 52:fe:54:6a:f8:fd brd ff:ff:ff:ff:ff:ff link-netnsid 0

root@container-foo:~# ip link set dev lo up
root@container-foo:~# ip link set dev veth_guest_foo up
root@container-foo:~# ip --br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
veth_guest_foo@if4 UP             fe80::50fe:54ff:fe6a:f8fd/64
## If you didn't bring the interface on the host up,
## you will see LOWERLAYERDOWN instead of UP.

## Note that routing table is empty inside the namespace.
## We need set it up.
##
## Recall the default routing rule on the host
##   default via 172.31.32.1 dev ens5 proto dhcp src 172.31.37.209 metric 100
root@container-foo:~# ip r
root@container-foo:~# 
root@container-foo:~# ip route add 172.31.32.1 dev veth_guest_foo
root@container-foo:~# ip route replace default via 172.31.32.1 dev veth_guest_foo

## Set the secondary IP for the namespace
root@container-foo:~# ip addr add 172.31.37.240 dev veth_guest_foo
root@container-foo:~# ip --br a
lo               UNKNOWN        127.0.0.1/8 ::1/128
veth_guest_foo@if4 UP             172.31.37.240/32 fe80::50fe:54ff:fe6a:f8fd/64

## It's not enough. We need let the interface know where to deliever frames at link layer.
## From the namespace (container) interface, the first hop will be the paired interface outside the namespace (host).
## Therefore, set the MAC address of veth_host_foo to gateway address (172.31.32.1)
root@container-foo:~# arp -i veth_guest_foo -s 172.31.32.1 66:29:2b:4d:aa:49
# MAC address matches the ip command result above run on the host
```

Now back to the host. We configured the host as the gateway router of the namespace (container). Hence, let us enable forwarding to let the kernel be able to work as a router. Also, we need route traffic to secondary IP that is used by `veth_guest_foo` into the namespace via the paired interface `veth_host_foo`.

```
# On the host

## Enable forwarding
ubuntu@foo:~$ sudo sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1

## Route traffic
ubuntu@foo:~$ sudo ip route add 172.31.37.240/32 dev veth_host_foo
```

Finally, we can ping from container to instances (`foo` and `bar`) in the VPC.
```
root@container-foo:~# ping -c 3 172.31.37.209  # foo
PING 172.31.37.209 (172.31.37.209) 56(84) bytes of data.
64 bytes from 172.31.37.209: icmp_seq=1 ttl=64 time=0.042 ms
64 bytes from 172.31.37.209: icmp_seq=2 ttl=64 time=0.036 ms
64 bytes from 172.31.37.209: icmp_seq=3 ttl=64 time=0.034 ms

--- 172.31.37.209 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2050ms
rtt min/avg/max/mdev = 0.034/0.037/0.042/0.003 ms

root@container-foo:~# ping -c 3 172.31.43.37 # bar
PING 172.31.43.37 (172.31.43.37) 56(84) bytes of data.
64 bytes from 172.31.43.37: icmp_seq=1 ttl=63 time=0.794 ms
64 bytes from 172.31.43.37: icmp_seq=2 ttl=63 time=0.163 ms
64 bytes from 172.31.43.37: icmp_seq=3 ttl=63 time=0.167 ms

--- 172.31.43.37 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2031ms
rtt min/avg/max/mdev = 0.163/0.374/0.794/0.296 ms
```

Similary, we can set up `container-bar` namespace in `bar`. After that, we can confirm the connectivity between containers.
```
## Ping from container-foo to container-bar
root@container-foo:~# ping -c3 172.31.37.27
PING 172.31.37.27 (172.31.37.27) 56(84) bytes of data.
64 bytes from 172.31.37.27: icmp_seq=1 ttl=62 time=0.460 ms
64 bytes from 172.31.37.27: icmp_seq=2 ttl=62 time=0.179 ms
64 bytes from 172.31.37.27: icmp_seq=3 ttl=62 time=0.246 ms

--- 172.31.37.27 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2032ms
rtt min/avg/max/mdev = 0.179/0.295/0.460/0.119 ms

## Ping from container-bar to container-foo
root@container-bar:~# ping -c3 172.31.37.240
PING 172.31.37.240 (172.31.37.240) 56(84) bytes of data.
64 bytes from 172.31.37.240: icmp_seq=1 ttl=62 time=0.330 ms
64 bytes from 172.31.37.240: icmp_seq=2 ttl=62 time=0.186 ms
64 bytes from 172.31.37.240: icmp_seq=3 ttl=62 time=0.173 ms

--- 172.31.37.240 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2025ms
rtt min/avg/max/mdev = 0.173/0.229/0.330/0.071 ms
```

### Egress to the Internet
We still have a "problem". We cannot access the Internet from the namespace even though we can do that on the host (Internet gateway is correctly configured for the host). Currently, we are using the secondary private IP inside the namespace (container). AWS will not translate it to public IP address.

```
root@container-foo:~# ping -c3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2043ms
```

Here is a solution. We can configure a SNAT to set the source IP to the primary IP when accessing non-VPC network.
```
# On the host

## Recall 172.31.37.209 is the primary IP address.
## 172.31.0.0/16 is our VPC CIDR.
ubuntu@foo:~$ sudo iptables -t nat -A POSTROUTING -o ens5 -j SNAT --to 172.31.37.209 ! -d 172.31.0.0/16
ubuntu@foo:~$ sudo iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
SNAT       all  --  0.0.0.0/0           !172.31.0.0/16        to:172.31.37.209
```

After doing that, we can access the Internet from the container.
```
root@container-foo:~# ping -c3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=101 time=3.26 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=101 time=3.25 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=101 time=3.35 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 3.252/3.288/3.351/0.044 ms
```

We don't discuss the ingress from the Internet since it's usually done by LB and LB can use IP addresses as members now.
{% include JB/setup %}

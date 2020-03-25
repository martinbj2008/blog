---
layout: post
title: "Add a Ip Address on a Interface(todo)"
date: 2012-08-29 00:00
comments: true
categories: [netcore]
tags: [kernel, route, network]
---

##summary
When a ip addr is added, two unicat route entries are added to route table.

1. a host route entry is added to local table.
  The packet to local host will be routed by this route entry. The route still is valid, even the related interface is shut down.
2. a connected route is added to main table.
  It is used to forward the packet to the hosts in the same sub network, it will disappear when interface down
3. two broad cast routes entry also added.

<!-- more -->
##broadcast route:

```
~ # dmesg  -c
~ # ifconfig 
~ # ip link
1: lo: <LOOPBACK> mtu 16436 qdisc noop state DOWN mode DEFAULT 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT 
    link/ether 16:83:71:72:da:97 brd ff:ff:ff:ff:ff:ff
3: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 08:00:27:89:78:8f brd ff:ff:ff:ff:ff:ff
4: eth1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 08:00:27:eb:21:53 brd ff:ff:ff:ff:ff:ff
5: eth2: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 08:00:27:2a:3f:e3 brd ff:ff:ff:ff:ff:ff
6: teql0: <NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 100
    link/void 
7: tunl0: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT 
    link/ipip 0.0.0.0 brd 0.0.0.0
8: sit0: <NOARP> mtu 1480 qdisc noop state DOWN mode DEFAULT 
    link/sit 0.0.0.0 brd 0.0.0.0
9: ip6tnl0: <NOARP> mtu 1452 qdisc noop state DOWN mode DEFAULT 
    link/tunnel6 :: brd ::
```
###ip link set eth1 up
```
~ # ip link set dev eth1 up
~ # dmesg  -c
IPv6: ADDRCONF(NETDEV_UP): eth1: link is not ready
e1000: eth1 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
IPv6: ADDRCONF(NETDEV_CHANGE): eth1: link becomes ready
~ # 
```
```
~ # ip route show table local
~ # ip route show table main
~ # dmesg  -c
~ # 
~ # ip addr add dev eth1 10.80.2.72/24    <=== configure a ip address on a interface.
~ # 
~ # dmesg -c <=== the related call trace.
inet_rtm_newaddr: is called
__inet_insert_ifa:  ifa=ffff88003d1fa7e0, ifa_address:4802500a
     ifa_local=4802500a,       ifa_mask=00ffffff
__inet_insert_ifa:  send rtm msg for ifa=ffff88003d1fa7e0
fib_inetaddr_event: event=00000001
fib_add_ifaddr: call fib_magic RTM_NEWROUTE, RTN_LOCAL with addr=4802500a
fib_magic: type=2, dst=4802500a, dst_len=32
fib_add_ifaddr: call fib_magic RTM_NEWROUTE, with frefixlen
fib_magic: type=1, dst=0002500a, dst_len=24
fib_add_ifaddr: call fib_magic RTM_NEWROUTE, to add two broadcast route
fib_magic: type=3, dst=0002500a, dst_len=32
fib_magic: type=3, dst=ff02500a, dst_len=32
```
###two broadcast routes are added.
```
~ # ip route list table local    <=== two broadcast routes are added to local, a host route(ip_local_in) is added also.
broadcast 10.80.2.0 dev eth1  proto kernel  scope link  src 10.80.2.72 
local 10.80.2.72 dev eth1  proto kernel  scope host  src 10.80.2.72 
broadcast 10.80.2.255 dev eth1  proto kernel  scope link  src 10.80.2.72 
a connected route is added.
```
```
~ # ip route list table main     <=== 
10.80.2.0/24 dev eth1  proto kernel  scope link  src 10.80.2.72    <=== it will be delete if interface down, while local route will not  be deleted.
set eth1 down,
```
```
~ # ip link set dev eth1 down
~ # ip route show table main <== nothing remain.
~ # ip route show table local
local 10.80.2.72 dev eth1  proto kernel  scope host  src 10.80.2.72  <== still there
ping localhost ip address with lo up
```
```
~ # ip link set dev lo up <== MUST confirm lo up.
~ # ping  -c 3 10.80.2.72  <== ping itself OK.
PING 10.80.2.72 (10.80.2.72): 56 data bytes
64 bytes from 10.80.2.72: seq=0 ttl=64 time=0.068 ms
64 bytes from 10.80.2.72: seq=1 ttl=64 time=0.069 ms
64 bytes from 10.80.2.72: seq=2 ttl=64 time=0.088 ms

--- 10.80.2.72 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.068/0.075/0.088 ms
~ # 
```

###related source:

in net/ipv4/devinet.c, register_inetaddr_notifier
```c
178 static BLOCKING_NOTIFIER_HEAD(inetaddr_chain);
1091 int register_inetaddr_notifier(struct notifier_block *nb)
net/ipv4/fib_frontend.c
```
```c
1190         register_inetaddr_notifier(&fib_inetaddr_notifier);
1089 static struct notifier_block fib_inetaddr_notifier = {
1090         .notifier_call = fib_inetaddr_event,
1091 };
737 void fib_add_ifaddr(struct in_ifaddr *ifa)
```

1. add a hostroute
2. add connect route
3. add two broadcast for network_address.255|0

net/ipv4/fib_frontend.c
```c
698 static void fib_magic(int cmd, int type, __be32 dst, int dst_len, struct in_ifaddr *ifa)
```

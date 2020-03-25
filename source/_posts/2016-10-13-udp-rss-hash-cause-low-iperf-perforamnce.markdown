---
layout: post
title: "udp rss hash cause low iperf perforamnce"
date: 2016-10-26 11:47:45 +0800
comments: true
categories: [netcore]
tags: [vxlan, rss, udp]
---

## vxlan下iperf性能问题
VXlan网络下，在两个容器(分别在两个host上)上，使用iperf进行tcp网络性能测试，带宽只能达到3.5Gb/s左右。
而两个容器所在的host机器之间是万兆网络环境，host上的网卡是ixgbe 10G网卡

## 解决方法
1. 发送端使用多线程参数 `-P`
 ```
 iperf  -c 192.168.51.2   -P8 -t 1000
 ```

2. 接收端IXGBE网卡RSShash使用hash(SrcIP, DstIP, <font color='red'>SrcPort, DstPort</font>)
 ```
 ethtool -N em2 rx-flow-hash udp4 sdfn
 ```
<!-- more -->

## 复现方法
在容器1启用server端：
```
iperf  -s
```

容器2上启用client端：
```
iperf  -c 192.168.51.2
```
## 问题分析

### ipef压测的同时，最好情况下可以看到4Gb/sec的带宽，带宽在3G左右波动
![图1](/images/udp_hash/udp_hash_1.png)

#### 使用top命令在容器所在的两台host机器上查看cpu利用率。
#### client端cpu很空闲的
![图2](/images/udp_hash/udp_hash_2.png)
#### server端cpu(cpu10)被压满了
![图3](/images/udp_hash/udp_hash_3.png)

所以单进程的性能已经接近瓶颈，因此使用并发多个线程。

 ```
 iperf  -c 192.168.51.2   -P8 -t 1000
 ```

 压测发现性能还是3-4G左右,效果提高不明显。

 top观察发现单核cpu的利用率奇高，已经被压爆了，但是其他cpu却没啥压力。
 因此怀疑rsshash没做好，

 ```
[root@bogon ~]# ethtool -n em2 rx-flow-hash udp4
UDP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA
 ```
 重新设置udp4的rsshash规则

 ```
[root@bogon ~]#  ethtool -N em2 rx-flow-hash udp4 sdfn
[root@bogon ~]# ethtool -n em2 rx-flow-hash udp4
UDP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA
L4 bytes 0 & 1 [TCP/UDP src port]
L4 bytes 2 & 3 [TCP/UDP dst port]
```
### 测试验证
iperf client使用4个线程压测
ifperf server端看到softirqd压力分不到了4个cpu上(cpu2,6,8,10)
![图5](/images/udp_hash/udp_hash_5.png)
总的带宽达到8.38Gb/s.
![图6](/images/udp_hash/udp_hash_6.png)

### 关于RSS可以参考这个图片及相应的解释

Packet Flow的图片

![RSS 介绍](http://balodeamit.blogspot.com/2013/10/receive-side-scaling-and-receive-packet.html)
http://2.bp.blogspot.com/-WRpAha4DF68/Ul9-j0IPCgI/AAAAAAAABjs/uHU5UuKrfnw/s1600/osi_layers+(6).png)

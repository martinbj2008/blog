---
layout: post
title: "WHAT is TFO"
date: 2016-11-23 18:56:08 +0800
comments: true
categories: 
---

## 什么是tcp fast open（TFO）
### 背景
网络上有大量的短连接，传输的数据很少。google统计显示，其访问请求里有30%左右的流量是短连接。     

### 原理
针对tcp 短连接的一个优化。在syn请求里携带请求数据，让server端尽早处理，进而降低一个RTT的延迟。     
client端发送时候在syn报文里，增加一个tcp option选项(TFO)，server端通过它校验client端的合法性。    

<!--more-->

普通tcp连接

![普通tcp连接](/images/tcp_3handshake.png)

TFO连接     
![TFO连接示意图](/images/tfo_generate.png)     

以http短连接wget为例:    

![TFO首次连接tcpdump](/images/tfo_generate_tcpdump.png)     
首次请求时候跟普通tcp类似，只是增加了一个TFO的tcp option。

首次连接过后,通过`ip tcp_metrics`可以查看到本地缓存的`TFO cookie`到
```
[root@localhost tfo_client]# ip tcp_metrics |grep "192\.168\.1\.2"
192.168.1.2 age 11626.653sec rtt 125us rttvar 250us cwnd 10 metric_5 795 metric_6 316 fo_mss 1460 fo_cookie 2880157680c6614d
```

![TFO后续连接](/images/tfo_handshake.png)     

client端syn请求在tcp option里携带协商好的cookie值，同时携带wget请求。
server端验证syn报文里的cookie，检验通过后，直接发起向nginx激发data ready,
处理wget请求，同时发送syn+ack（如果处理足够快，还会携带wget的相应数据）。
![TFO后续连接tcpdump](/images/tfo_handshake_tcpdump.png)     

注
+ 如果client携带是一个错误的TFO cookie值，该连接会被当做普通的tcp连接对待，同时server回复的`sync_ack`报文里会携带一个正确的TFO cookie。
+ 使用命令`ip tcp_metrics delete`可以强制清除缓存的TFO cookie

## TFO如何使用
### 在sysctl里把tfo打开
```
[root@vmcentos7 nginx-1.10.2]# echo 3 > /proc/sys/net/ipv4/tcp_fastopen
[root@vmcentos7 nginx-1.10.2]# cat /proc/sys/net/ipv4/tcp_fastopen
3
```
这里`3`是失能TFO的全部功能，也可以根据实际需求仅使能client端或者server端。
```
 221 /* Bit Flags for sysctl_tcp_fastopen */
 222 #define TFO_CLIENT_ENABLE       1
 223 #define TFO_SERVER_ENABLE       2
 224 #define TFO_CLIENT_NO_COOKIE    4       /* Data in SYN w/o cookie option */
```


### server端的改动
 + sourcecode
```
	int err = setsockopt(sfd, IPPROTO_TCP/*SOL_TCP*/, 23/*TCP_FASTOPEN*/, &qlen, sizeof(qlen));
```
 + 使用nginx作为server端
重新编译nginx，configure时候增加TFO的选项
```
-DTCP_FASTOPEN=23'
```
### client端的改动
 不再调用`connect`， 同时将send调用改为sendto
```
 	int len = sendto(sfd, data, data_len, 0x20000000/*MSG_FASTOPEN*/,
			(struct sockaddr *) &serv_addr, sizeof(serv_addr));
```
## TFO在内核里的实现

## TFO性能测试ht-is-tfo.markdown
###  延迟测试
两台物理机在同一个交换机下，使用万兆网口互联，无压力下测试。
循环wget 一个test.html测试页，10000次，平均每次的时间差别30us左右。
```
tcp_test average 119 us, 10000 loops
tfo_test average 89 us, 10000 loops
```

## 支持TFO的系统
 centos 7.2 内核已经支持TFO.

+ IPv4 support for TFO was merged into the Linux kernel mainline in kernel versions 3.6 (support for clients) and 3.7 (support for servers), 
and was turned on by default in kernel version 3.13. 
TFO support for IPv6 servers was merged in kernel version 3.16.
+ Google Chrome and Chromium browsers have support for TFO on Linux, including Chrome OS and Android.
+ Apple's iOS 9 and OS X 10.11 both support TCP Fast Open, but it is not enabled for individual connections by default.[9]
+ Microsoft Edge supports TCP Fast Open since Windows 10 Preview build 14352.[10]

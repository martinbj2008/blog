---
layout: post
title: "how to remember ip/tcp/udp header"
date: 2014-01-27 14:25
comments: true
categories: [socket]
---

### IP HEADER
以前找工作时，常被问到IP头都有哪些字段？
现在觉得真的理解了记起来没那么难。

上路时总要记得终点和起点(src/dst ip),
要倒几次车，大体也知道（ttl).

路有大路有小路，有高速路和土路。
大路到小路要分片，小路到大路可能重组。
上高速路需要通行证（qos/tos).

路上有警察，查你是不是非法(csum, header len, ip option)，
查你装的什么货(protocol)？

<!-- more -->
1. 路由：要做路由，IP地址是必须的, 如何防止循环链路？
	 源/目的IP地址，ttl
2. 分片与重组：ID, 总长度，片的偏移，标志位,
	ID, total length, offset, E/D/M flags
3. QOS:  TOS
4. 校验：checksum
5. 上层协议：proto
![case 1](/images/net/ip.header.png)
### TCP HEADER
![case 1](/images/net/tcp.header.png)
### UDP HEADER
![case 1](/images/net/udp.header.png)
### ICMP HEADER
![case 1](/images/net/icmp.header.png)

NOTE: all the pictures are 'stolen' from
`http://nmap.org/book/tcpip-ref.html`

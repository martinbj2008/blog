---
title: net rx drop
date: 2020-03-25 21:16:54
tags:
---

## 问题来源
接OP问题报告，监控到部分机器的net rx drop统计值异常，触发报警，
需排查具体原因，并确认是否影响业务。

## 问题分析
### 复现问题 
跟OP同学确认,通过采集`/proc/net/dev`下的rx drop。
登录到出问题的机器上, 确认内核该统计值确实异常。
+ 异常报文个数不多，大约1s一个左右。
+ 不是所有机器都有异常，有部分机器drop统计为0.

### 相关内核代码
当一个数据报文经过对端设备（交换机或者网卡）传输到本机物理网卡时候，需要经过 
    `网卡-- 网卡驱动-- 网络协议栈` 
这几个模块的处理。我们看到的drop统计值在增加，是在网络协议栈的入口处理部分产生异常导致。

<!--more-->

#### 内核网络协议栈（伪）入口代码：
注：本文里使用内核3.10.0-514.16.1代码

`net/core/dev.c`
```
3709 static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
3710 {
....
3753
3754         list_for_each_entry_rcu(ptype, &ptype_all, list) {
3755                 if (!ptype->dev || ptype->dev == skb->dev) {
3756                         if (pt_prev)
3757                                 ret = deliver_skb(skb, pt_prev, orig_dev);
3758                         pt_prev = ptype;
3759                 }
3760         }
3761
....
3814
3815         /* deliver only exact match when indicated */
3816         null_or_dev = deliver_exact ? skb->dev : NULL;
3817
3818         type = skb->protocol;
3819         list_for_each_entry_rcu(ptype,
3820                         &ptype_base[ntohs(type) & PTYPE_HASH_MASK], list) {
3821                 if (ptype->type == type &&
3822                     (ptype->dev == null_or_dev || ptype->dev == skb->dev ||
3823                      ptype->dev == orig_dev)) {
3824                         if (pt_prev)
3825                                 ret = deliver_skb(skb, pt_prev, orig_dev);
3826                         pt_prev = ptype;
3827                 }
3828         }
3829
3830         if (pt_prev) {
3831                 if (unlikely(skb_orphan_frags(skb, GFP_ATOMIC)))
3832                         goto drop;
3833                 else
3834                         ret = pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
3835         } else {
3836 drop:
3837                 if (!deliver_exact)
3838                         atomic_long_inc(&skb->dev->rx_dropped);
3839                 else
3840                         atomic_long_inc(&skb->dev->rx_nohandler);
3841                 kfree_skb(skb);
3842                 /* Jamal, now you will not able to escape explaining
3843                  * me how you were going to use this. :-)
3844                  */
3845                 ret = NET_RX_DROP;
3846         }
3847
3848 out:
3849         return ret;
3850 }
```
有同学可能质疑rx_dropped的统计值不止这一处在修改（增加), 内核里确实不止这一处在增加drop。
这里使用perf调试命令，可以确认3838行的增加跟总统计值匹配，而不是其他地方导致增加。
具体perf命令的使用见下一节。

### IPv6报文定位
在使用perf命令查看统计值增长点的时候，我们搂草打兔子，捎带把skb的protocol给打印出来了。
截止到目前可以认为protocol的值就是我们收的网络报文的ethernet头里的protocol字段。

```
[root@testtest ~]# perf probe -a "__netif_receive_skb_core:129 skb->protocol"
Added new events:
  probe:__netif_receive_skb_core (on __netif_receive_skb_core:129 with protocol=skb->protocol)
  probe:__netif_receive_skb_core_1 (on __netif_receive_skb_core:129 with protocol=skb->protocol)

You can now use it in all perf tools, such as:

	perf record -e probe:__netif_receive_skb_core_1 -aR sleep 1

[root@testtest ~]# perf record -e probe:__netif_receive_skb_core_1 -aR sleep 5
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.209 MB perf.data (5 samples) ]
[root@testtest ~]#
[root@testtest ~]#
[root@testtest ~]# perf script
         swapper     0 [003] 596928.567038: probe:__netif_receive_skb_core_1: (ffffffff8152afb6) protocol=dd86
         swapper     0 [011] 596929.277293: probe:__netif_receive_skb_core_1: (ffffffff8152afb6) protocol=400
         swapper     0 [001] 596929.431077: probe:__netif_receive_skb_core_1: (ffffffff8152afb6) protocol=dd86
         swapper     0 [008] 596930.073356: probe:__netif_receive_skb_core_1: (ffffffff8152afb6) protocol=dd86
         swapper     0 [011] 596931.278891: probe:__netif_receive_skb_core_1: (ffffffff8152afb6) protocol=400
[root@testtest ~]#
```

看到`protocol=dd86`，drop增加的触发原因就找到了（一个）。

因为我们公司是把内核的IPv6功能禁止的，网络协议栈收到IPv6报文的时候是不做任何处理并丢弃的。
```
[root@testtest ~]# cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-3.10.0-327.36.3.el7.x86_64 root=UUID=7918e921-db4e-4652-9938-a38607100a96 ro crashkernel=128M selinux=0 ipv6.disable=1
```

为了进一步验证分析结果，上一个tcpdump的结果
```
[root@testtest ~]# tcpdump -i eth0 ip6 -c 10
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
14:15:40.663327 IP6 fe80::6e0b:84ff:fe81:b4d3.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:15:45.334814 IP6 fe80::6e0b:84ff:fed6:c323.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:15:45.403538 IP6 fe80::6e0b:84ff:febd:aef8.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:15:52.542434 IP6 fe80::6e0b:84ff:fe81:b935.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:16:00.186710 IP6 fe80::6e0b:84ff:febd:b27c.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:16:06.397437 IP6 fe80::6e0b:84ff:fe81:ba03.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:16:07.834059 IP6 fe80::6e0b:84ff:febd:b602.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:16:22.138601 IP6 fe80::6e92:bfff:fe2e:616f.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:16:31.827770 IP6 fe80::6e0b:84ff:fed6:c053.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
14:16:31.882039 IP6 fe80::6e0b:84ff:febd:b798.dhcpv6-client > ff02::1:2.dhcpv6-server: dhcp6 solicit
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[root@testtest ~]#
```

注：
1. 这里的截屏是写文章时候去线上测试机抓取的，部分机器上的IPv6问题被修复过一部分。
所以，我们看到的IPv6报文个数比当初分析问题的时候，相对要偏少。

IPv6报文的来源：服务器同学定位是浪潮服务器的网卡主动发出的。

### STP报文定位
#### 异常报文统计有出入

在使用perf工具定位IPv6报文后，人肉查看，发现IPv6的个数大约占drop统计值的60%左右，
也就是说，还有其他类型的报文导致drop统计值在增加。
在硬件厂商修复网卡问题后，观察到大幅降低了drop的增长速度，但没有彻底消除drop的异常统计增长。

正如perf结果显示还有部分`protocol=400`的报文，这是些啥报文？

我们搜集到了如下一些信息：  
+ tcpdump全量抓包，也没有看到`400`的报文。
+ 查询RFC相关定义，可以发现他们是`802.2 LLC`
+ 在内核代码里，被网桥的llc模块处理的。

##### 内核代码

`net/llc/llc_core.c`

```
110 #define ETH_P_802_2     0x0004          /* 802.2 frames                 */

135 static struct packet_type llc_packet_type __read_mostly = {
136         .type = cpu_to_be16(ETH_P_802_2),
137         .func = llc_rcv,
138 };
```

做一个试验：
    手动加载llc模块，加载后drop统计值不再增张了。
这个实验侧面验证了，有部分网桥相关的报文被传递到内核网络协议栈了。

##### `802.2 LLC`
LLC对应了一大家族的链路协议类型，我们需要定位具体是哪一种类型的报文引起的。
这就需要把`protocol=400`的报文被统计异常的时候，把它当时对应的ethernet头信息打印出来。
针对这样的需求，下一节的centos7上的`核弹`（热补丁）派上用场了。

注：
    LLC对应一大坨的链路层协议，网卡驱动（以ixgbe为例）会借助`eth_type_trans`函数，
    把这些ethernet 协议汇总为`802.2 LLC`，并存放在`skb->protocol`里的。
    
`drivers/net/ethernet/intel/ixgbe/ixgbe_main.c`

```
1665 static void ixgbe_process_skb_fields(struct ixgbe_ring *rx_ring,
1666                                      union ixgbe_adv_rx_desc *rx_desc,
1667                                      struct sk_buff *skb)
1668 {
1669         struct net_device *dev = rx_ring->netdev;
1670         u32 flags = rx_ring->q_vector->adapter->flags;
1671
1672         ixgbe_update_rsc_stats(rx_ring, skb);
1673
1674         ixgbe_rx_hash(rx_ring, rx_desc, skb);
1675
1676         ixgbe_rx_checksum(rx_ring, rx_desc, skb);
1677
1678         if (unlikely(flags & IXGBE_FLAG_RX_HWTSTAMP_ENABLED))
1679                 ixgbe_ptp_rx_hwtstamp(rx_ring, rx_desc, skb);
1680
1681         if ((dev->features & NETIF_F_HW_VLAN_CTAG_RX) &&
1682             ixgbe_test_staterr(rx_desc, IXGBE_RXD_STAT_VP)) {
1683                 u16 vid = le16_to_cpu(rx_desc->wb.upper.vlan);
1684                 __vlan_hwaccel_put_tag(skb, htons(ETH_P_8021Q), vid);
1685         }
1686
1687         skb_record_rx_queue(skb, rx_ring->queue_index);
1688
1689         skb->protocol = eth_type_trans(skb, dev);
1690 }
```

`net/ethernet/eth.c`

```
188 __be16 eth_type_trans(struct sk_buff *skb, struct net_device *dev)
189 {
...
230         if (likely(eth_proto_is_802_3(eth->h_proto)))
231                 return eth->h_proto;
...
242         /*
243          *      Real 802.2 LLC
244          */
245         return htons(ETH_P_802_2);
246 }
247 EXPORT_SYMBOL(eth_type_trans);
```

#### centos7上的`核弹` 内核热升级工具
centos7（3.10内核）开始，内核组已经部署了一个叫`centos-kpatch-modules`的rpm包，
它可以做到业务不感知的情况下，加载调试模块，调试完就可卸载。

刚好出问题的机器内核版本是`3.10.0-514.16.1.el7.x86_64`，
于是编写了一个调试模块(后续会集成到`centos-kpatch-modules`)，其功能是：
    当drop掉异常报文的时候，把报文的内容（ethernet头）打印出来。

通过kpatch命令加载调试模块，就能把被drop的报文头信息打印出来，

```
[root@testtest xfile]# uname -a
Linux testtest 3.10.0-327.36.3.el7.x86_64 #1 SMP Mon Oct 24 16:09:20 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
[root@testtest xfile]# rpm -q centos-kpatch-modules
centos-kpatch-modules-1.6-1.x86_64
[root@testtest xfile]# kpatch  load /root/martin/xfile/net_rx_drop/327/kpatch-dev.ko
loading patch module: /root/martin/xfile/net_rx_drop/327/kpatch-dev.ko
[root@testtest xfile]# dmesg
[606774.224447] kpatch: loaded patch module 'kpatch_dev'
[606775.135343] __netif_receive_skb_core: skb len:105 protocol:0004
[606775.135348] skb_mac_head:ffff88003625c440 skb data:ffff88003625c44e, mac hdr offset:14
[606775.135350] mac hdr dest:01:80:c2:00:00:00 <=== 50:da:00:ef:54:0a proto: 0069
[606775.337246] __netif_receive_skb_core: skb len:96 protocol:86dd
[606775.337252] skb_mac_head:ffff880850046c40 skb data:ffff880850046c4e, mac hdr offset:14
[606775.337255] mac hdr dest:33:33:00:01:00:02 <=== 6c:0b:84:81:ba:03 proto: 86dd
[root@testtest xfile]#
```

通过日志我们看到被drop的报文ethernet头里protocol字段是`0069`, 其转化后存在skb里的protocol字段是`0004`。

tcpdump命令也显示有报文进入(此处说法不严谨，后续会被打脸)。
```
[root@localhost ~]# tcpdump -i em1 not ip -c 8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on em1, link-type EN10MB (Ethernet), capture size 65535 bytes
04:55:07.868905 IP6 fe80::303b:6dff:fe6a:20c4 > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
04:55:08.183959 STP 802.1d, Config, Flags [none], bridge-id 812c.38:1c:1a:7a:c1:80.8036, length 42
04:55:08.738143 STP 802.1s, Rapid STP, CIST Flags [Learn, Forward, Agreement], length 102
04:55:08.790157 IP6 fe80::5c11:3ff:fe55:9037 > ff02::2: ICMP6, router solicitation, length 8
04:55:08.914911 IP6 :: > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
04:55:09.033921 IP6 :: > ff02::16: HBH ICMP6, multicast listener report v2, 1 group record(s), length 28
04:55:09.185917 IP6 :: > ff02::1:ffab:f174: ICMP6, neighbor solicitation, who has fe80::18cc:d5ff:feab:f174, length 24
04:55:09.773337 IP6 fe80::e818:a4ff:fe30:c522 > ff02::2: ICMP6, router solicitation, length 8
8 packets captured
14 packets received by filter
0 packets dropped by kernel
[root@localhost ~]#
```
至此，我们锁定STP报文是drop统计增加的另一个来源。

综合多台机器测试结果，被丢弃的报文有三类
+ IPv6
+ STP
+ LLDP

至此问题上半部分析清楚。

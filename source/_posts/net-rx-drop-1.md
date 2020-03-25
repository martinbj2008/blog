---
title: net rx drop(续)
date: 2020-03-25 21:20:02
tags:
---

### 接上部
如上半部分析，导致rx drop值增加的报文有三类

+  IPv6
+  STP
+  LLDP

至此，为什么drop统计值增加，能够解释清楚了。

<!--more-->

扫描业务同学的机器列表这些drop异常机器都是浪潮的服务器，且机器网卡信息如下：
```
[root@localhost ~]# ethtool -i em1
driver: ixgbe
version: 4.4.0-k-rh7.3
firmware-version: 0x800007f4
...
```

跟网络和服务器的同学勾兑后，发现还有几个问题未解释清楚：

+ Q1: centos6上未发现此类问题
+ Q2: STP报文线上所有交换机都是打开的，但只有部分centos7的机器上部分机器出现这个问题。
+ Q3: 软硬件（内核版本，网卡，驱动，firmware-version）完全相同的浪潮服务器的机器，一台有drop统计，另外一台没有
+ Q4: tcpudmp命令执行的时候，drop统计值停止增长。

接下来要依次排查并解释以上几个问题。

## Q1: 为啥Centos6上drop没有异常
centos6(`rhel-2.6.32-573.el6`)的代码如下，
原因：没有统计这种错误。

```
3268 int __netif_receive_skb(struct sk_buff *skb)
3269 {
...
3386         if (pt_prev) {
3387                 ret = pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
3388         } else {
3389                 kfree_skb(skb);
3390                 /* Jamal, now you will not able to escape explaining
3391                  * me how you were going to use this. :-)
3392                  */
3393                 ret = NET_RX_DROP;
3394         }
```

## Q2: 为啥只有部分centos7的机器上部分机器出现这个问题

答案：因为STP报文被部分网卡drop掉，所以统计值没有异常。出问题的机器上STP报文被转发给内核了。

我们进行3个测试：

### 测试1：使用热升级调试模块
因为我们之前使用该模块定位了STP导致drop值增加，所以又在没有增加的机器加载调试模块。
结果发现，没有任何打印输出，确认内核协议栈没有收到STP报文。

### 测试2：查看网卡报文统计，没有组播报文
```
root@XXXX:~$ ethtool  -S eth0|grep multicast
     multicast: 140
```
该统计值不会增加
注：
    这里非零是因为我们这台机器上做了后续的试验导致的。

### 测试3：网卡设置为混杂模式
打开网卡混杂模式，原本net rx drop没有变化的机器， 统计值也开始增加了。
关闭网卡混杂模式， 统计值不再变化。
```
root@XXXX:~$ ip link set dev eth0 promisc on

root@XXXX:~$ cat /proc/net/dev;sleep 10; !cat
cat /proc/net/dev;sleep 10; cat /proc/net/dev
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
  eth0: 1890541692760 3759090874    0   26    0     0          0       102 1702993907316 3031053938    0    0    0     0       0          0

##10s后（26-->32)  
 eth0: 1890543854444 3759103725    0   32    0     0          0       108 1702995106816 3031063115    0    0    0     0       0          0
 root@XXXX:~$
```

通过上面的几个试验：
+ 测试1：可以确认网络协议栈没有收到stp报文。
+ 测试2：可以确认网卡没处理STP报文
+ 测试3：使用混杂模式，强制使STP报文发到协议栈后，统计值就增加了。

注：不要通过tcpdump去设置混杂模式，tcpdump可以让网卡处于混杂模式，但会导致net rx drop不变。
具体原因后续分析。

## Q3: 软硬件相同的机器，一台有drop统计，另外一台没有

在没有问题的机器上，查看`ptye`信息我们发现，内核的`ptype_all`被加载了一个的
处理函数`tpacket_rcv`.

```
root@YYYYY:~$ cat /proc/net/ptype
Type Device      Function
ALL           tpacket_rcv
0800          ip_rcv
0806          arp_rcv
```
`tpacket_rcv`有的同学可能不了解，但是tcpdump/libpcap，我们经常是经常使用的。
他们会依赖内核的的`tpacket_rcv`。
当我们需要从内核抓取全量或者特定类型的报文时候，`tpacket_rcv`经常被用到。

### `ptype_all` vs `ptype_base`

Linux内核维护了两个函数指针数组（不严谨，类比），`ptype_all` vs `ptype_base`
`ptype_base`： 相对比较容易理解。 当需要处理某种ethernet报文的时候，我们就注册一个
钩子函数到`ptype_base`。比如，
+ 内核使用`ip_rcv`处理IPv4报文。
+ `arp_rcv`处理ARP报文
+ `ipv6_rcv`处理IPv6报文。

`ptype_all`：是用来做报文分流镜像的， 比如我们使用tcpudmp命令，附加特定过滤条件，
抓取内核协议栈收到的报文。
这时候会在`ptype_all`里注册一个钩子函数，将报文导入到对应的`af_pack`模块里进行
处理（tcpdump的部分filter功能就在这个模块里执行）。

`ptype_base`是针对特定的协议，而`ptype_all`不做区分具体协议类型。
这样也就可以解释了Q4关于tcpudmp的问题了。

## Q4: tcpudmp命令执行的时候，drop统计值停止增长。
见Q3.

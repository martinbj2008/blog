---
layout: post
title: "how tcpdown direction filter work"
date: 2015-05-22 11:51:57 +0800
comments: true
categories: [netcore]
tags: [tcpdump]
---

## tcpudmp对 direction的支持。
内核代码有一个关键数据结构：skb的pkt_type字段。
在收发路径这个域被赋值为PACKET_OUTGOING或者其他。
这个值被传递到往用户空间，libpcap根据它判断报文的方向是否是期望的。

### pkt_type的可能取值 
```
 24 #define PACKET_HOST             0               /* To us                */
 25 #define PACKET_BROADCAST        1               /* To all               */
 26 #define PACKET_MULTICAST        2               /* To group             */
 27 #define PACKET_OTHERHOST        3               /* To someone else      */
 28 #define PACKET_OUTGOING         4               /* Outgoing of any type */
```
<!-- more -->

kernel 里对网络包的抓去分别在收方两个方向做分析。
#### 收方向
```
> netif_receive_skb
> > netif_receive_skb_internal
> > > __netif_receive_skb
> > > > __netif_receive_skb_core
> > > > > list_for_each_entry_rcu(ptype, &ptype_all, list) {
```

### 发方向 :dev_queue_xmit_nit 具体调用关系如下
```
> dev_hard_start_xmit(struct
> > xmit_one
> >  > dev_queue_xmit_nit
> > > > list_for_each_entry_rcu(ptype, ptype_list, list)
> > > > > skb2->pkt_type = PACKET_OUTGOING;
```

#### tcpudmp注册hook函数到ptye_all里的
packet socket在创建时候已经制定了处理函数为packet_rcv，并注册到ptype_all里。
```
2808 static int packet_create(struct net *net, struct socket *sock, int protocol,
...
2868         po->prot_hook.func = packet_rcv;
2869
2870         if (sock->type == SOCK_PACKET)
2871                 po->prot_hook.func = packet_rcv_spkt;
2872
2873         po->prot_hook.af_packet_priv = sk;
2874
2875         if (proto) {
2876                 po->prot_hook.type = proto;
2877                 register_prot_hook(sk);
2878         }
```

```
 337 static void register_prot_hook(struct sock *sk)
 338 {
 339         struct packet_sock *po = pkt_sk(sk);
 340
 341         if (!po->running) {
 342                 if (po->fanout)
 343                         __fanout_link(sk, po);
 344                 else
 345                         dev_add_pack(&po->prot_hook);
 346
 347                 sock_hold(sk);
 348                 po->running = 1;
 349         }
 350 }
```

#### hook函数｀packet_rcv`会将pkt_type传递到用户空间
```
1766 static int packet_rcv(struct sk_buff *skb, struct net_device *dev,
1767                       struct packet_type *pt, struct net_device *orig_dev)
1768 {
...
1831         sll->sll_pkttype = skb->pkt_type;
```

#### libpcap根据pkt_type／sll_pkttype的数值判断方向是否合法。
```
1433 static inline int
1434 linux_check_direction(const pcap_t *handle, const struct sockaddr_ll *sll)
1435 {
1436         struct pcap_linux       *handlep = handle->priv;
1437
1438         if (sll->sll_pkttype == PACKET_OUTGOING) {
1439                 /*
1440                  * Outgoing packet.
1441                  * If this is from the loopback device, reject it;
1442                  * we'll see the packet as an incoming packet as well,
1443                  * and we don't want to see it twice.
1444                  */
1445                 if (sll->sll_ifindex == handlep->lo_ifindex)
1446                         return 0;
1447
1448                 /*
1449                  * If the user only wants incoming packets, reject it.
1450                  */
1451                 if (handle->direction == PCAP_D_IN)
1452                         return 0;
1453         } else {
1454                 /*
1455                  * Incoming packet.
1456                  * If the user only wants outgoing packets, reject it.
1457                  */
1458                 if (handle->direction == PCAP_D_OUT)
1459                         return 0;
1460         }
1461         return 1;
1462 }
```


#### 注:upstream的内核开始支持per netdevice的type chain

```
dev: add per net_device packet type chains 
commit 7866a621043fbaca3d7389e9b9f69dd1a2e5a855
Author: Salam Noureddine <noureddine@arista.com>
Date:   Tue Jan 27 11:35:48 2015 -0800

    dev: add per net_device packet type chains
```

```
packet_do_bind
> static int packet_do_bind(struct sock *sk, struct net_device *dev, __be16 proto)
>  > register_prot_hook(sk);
>  >  >  dev_add_pack(&po->prot_hook);
>  >  >  > struct list_head *head = ptype_head(pt); <== return the ptye._all 或者对应设备的ptye-all
>  >  >  >  398         list_add_rcu(&pt->list, head);
```

```
 371 static inline struct list_head *ptype_head(const struct packet_type *pt)
 372 {
 373         if (pt->type == htons(ETH_P_ALL))
 374                 return pt->dev ? &pt->dev->ptype_all : &ptype_all;
 375         else
 376                 return pt->dev ? &pt->dev->ptype_specific :
 377                                  &ptype_base[ntohs(pt->type) & PTYPE_HASH_MASK];
 378 }
```


---
layout: post
title: "TCP ack study"
date: 2016-02-26 09:35:58 +0800
comments: true
categories: 
---
kernel version `v4.5`    

两个重要并且容易混淆的函数:   
* `tcp_v4_rcv`    
* `tcp_v4_do_rcv`   

类似于中断处理的上半部和下半部，
tcp的处理分为了的总入口函数是`tcp_v4_rcv`,
而`tcp_v4_do_rcv`则是真正处理tcp报文,
并传送到用户空间。

其他的像拥塞控制，乱序调整等都在`tcp_v4_do_rcv`之前被做掉了。

<!-- more -->

```
1506 static const struct net_protocol tcp_protocol = {
1507         .early_demux    =       tcp_v4_early_demux,
1508         .handler        =       tcp_v4_rcv,
1509         .err_handler    =       tcp_v4_err,
1510         .no_policy      =       1,
1511         .netns_ok       =       1,
1512         .icmp_strict_tag_validation = 1,
1513 };
```


```
1539 int tcp_v4_rcv(struct sk_buff *skb)
1540 {
...
转换 tcp头部信息并保存到 TCP_SKB_CB
1580         TCP_SKB_CB(skb)->seq = ntohl(th->seq);
1581         TCP_SKB_CB(skb)->end_seq = (TCP_SKB_CB(skb)->seq + th->syn + th->fin +
1582                                     skb->len - th->doff * 4);
1583         TCP_SKB_CB(skb)->ack_seq = ntohl(th->ack_seq);
1584         TCP_SKB_CB(skb)->tcp_flags = tcp_flag_byte(th);
1585         TCP_SKB_CB(skb)->tcp_tw_isn = 0;
1586         TCP_SKB_CB(skb)->ip_dsfield = ipv4_get_dsfield(iph);
1587         TCP_SKB_CB(skb)->sacked  = 0;

查找对应的tcp socket
1589 lookup:
1590         sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest);
1591         if (!sk)
1592                 goto no_tcp_socket;

如果是tcp server:(LISTEN socket):
1643         if (sk->sk_state == TCP_LISTEN) {
1644                 ret = tcp_v4_do_rcv(sk, skb);
1645                 goto put_and_return;
1646         }


一般的的tcp连接:
如果正在被用户锁定，那么就进入backlog队列。
否则则尝试进入prequeue队列。如果进入队列失败，
则直接调用 tcp_v4_do_rcv,做tcp的下半段处理。
1650         bh_lock_sock_nested(sk);
1651         tcp_sk(sk)->segs_in += max_t(u16, 1, skb_shinfo(skb)->gso_segs);
1652         ret = 0;
1653         if (!sock_owned_by_user(sk)) {
1654                 if (!tcp_prequeue(sk, skb))
1655                         ret = tcp_v4_do_rcv(sk, skb);
1656         } else if (unlikely(sk_add_backlog(sk, skb,
1657                                            sk->sk_rcvbuf + sk->sk_sndbuf))) {
1658                 bh_unlock_sock(sk);
1659                 NET_INC_STATS_BH(net, LINUX_MIB_TCPBACKLOGDROP);
1660                 goto discard_and_relse;
1661         }
1662         bh_unlock_sock(sk);
1663
1664 put_and_return:
1665         sock_put(sk);
1666
1667         return ret;
```

```
```

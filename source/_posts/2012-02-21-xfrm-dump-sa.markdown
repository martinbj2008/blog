---
layout: post
title: "xfrm dump sa"
date: 2012-02-10 00:00
comments: true
categories: [xfrm]
tags: [kernel, xfrm, dump sa]
---

##应用层通过pfkey，dump sa的步骤：

1. 创建pfkeyl类型的一个socket， 
2. 通过sendmsg发送一个dump sa 的请求 
3. 然后通过while循环调用recvmsg

<!-- more -->
##kernel里对应的处理：

内核遍历存放SA的list。将SA的信息转化到一个或多个skb, 将这些skb添加到相应pfkey socket的sk_receive_queue链表里。
每次添加Skb时候会检查是否超出socket允许接受的最大receive字节数。
如果SA的数量比较多,则无法一次将所有的SA信息转化到skb里。


当SA的个数比较少时，一次性输出所有的SA是可以的， 但是当SA的个数非常大时， 一次性输出（遍历） 所有的sa是低效的(消耗大量内存），而且还要受到输出缓冲区buff的限制。

因此xfrm walk 采用分段遍历的方式，每次只顺着`net->xfrm.state_all`链表往后推进一步， 这样一次一步逐渐分批次把真个list的SA全部遍历完毕。 而`struct xfrm_state_walk`这个结构体就是用来记录当前这次遍历的位置。

具体实现就是将`xfrm_state_walk`这个结构体插入到`net->xfrm.state_all`这个hlisst上， 并位于最近一次遍历到的SA的后边。

在下一次继续遍历的时候,则从xfrm_state_walk这个结构体继续往后走， 如果遍历完全部SA，则把`xfrm_state_walk`这个结构体从`net->xfrm.state_all`上删除， 否则移动`xfrm_state_walk`这个结构体到本次遍历的最有一个SA的后面，并等待下次继续遍历。 如此反复，可以遍历完所有的SA, 最后删除xfrmwalk。

这是netlink callback机制在内核里的一个具体的实现。还有很多netlink callback的例子。 
xfrm state采用的是个单链表。有个callback机制使用的是hlist.

无论hlist还是list，在dump时，是加锁的， 可以保证数据的一致性。
但是当数据量大的时候，callback机制无法保证，两次dump的过程中会有增删的操作。
因此应用层程序，可以有两种方式去解决这个问题：

1. 在dump的同时监听相应的增删消息。
2. 判断netlink消息里是否有标志。如果有则重新dump。

### calltrace

```c
== err = pfkey_process(sk, skb, hdr);
== == err = pfkey_funcs[hdr->sadb_msg_type](sk, skb, hdr, ext_hdrs);
== == == pfkey_dump
== == == == pfkey_do_dump
== == == == == pfk->dump.msg_pid = hdr->sadb_msg_pid;
== == == == == pfk->dump.dump = pfkey_dump_sa;
== == == == == pfk->dump.done = pfkey_dump_sa_done;
== == == == == xfrm_state_walk_init(&pfk->dump.u.state, proto);
== == == return pfkey_do_dump(pfk);

##Data structure

1. 每个net都有一个netns_xfrm结构体，里面保存了xfrm的信息。

```c
35 struct net {
...
87 #ifdef CONFIG_XFRM
88         struct netns_xfrm       xfrm;
89 #endif
```

2. netns_xfrm下有一个state_all的list（双向循环）挂这改net下的所有的xfrm state。

```c
17 struct netns_xfrm {
18         struct list_head        state_all;
```

3. 每个xfrm state有一个 xfrm_state_walk类型的km， xfrm_state_walk里有个链all 最终每个xfrm state通过 km.all 挂在 net->xfrm.state_all这条链上。

```
 129 struct xfrm_state {
....
 149         /* Key manager bits */
 150         struct xfrm_state_walk  km; 
```

```c
 118 struct xfrm_state_walk {
 119         struct list_head        all;
 120         u8                      state;
 121         union {
 122                 u8              dying;
 123                 u8              proto;
 124         };
 125         u32                     seq;
 126 };      
```

因此遍历net->xfrm.state_all可以得到改net下的全部SA。

##functions
1. `pfkey_do_dump`

```c
 279 static int pfkey_do_dump(struct pfkey_sock *pfk)
 280 {
 281         struct sadb_msg *hdr;
 282         int rc;
 283        
 284         rc = pfk->dump.dump(pfk);
 285         if (rc == -ENOBUFS)
 286                 return 0;
 287 
 288         if (pfk->dump.skb) {
 289                 if (!pfkey_can_dump(&pfk->sk))  
 290                         return 0;
 291  
 292                 hdr = (struct sadb_msg *) pfk->dump.skb->data;
 293                 hdr->sadb_msg_seq = 0;
 294                 hdr->sadb_msg_errno = rc;
 295                 pfkey_broadcast(pfk->dump.skb, GFP_ATOMIC, BROADCAST_ONE,
 296                                 &pfk->sk, sock_net(&pfk->sk));
 297                 pfk->dump.skb = NULL;
 298         }
 299 
 300         pfkey_terminate_dump(pfk);
 301         return rc;
 302 }
```
相当于pfkey_dump_sa
```c
== 针对SA 每个sa调用
== == dump_sa
== == == pfkey_broadcast_one
== == == == skb_queue_tail(&sk->sk_receive_queue, *skb2);
xfrm_state_walk(net, &pfk->dump.u.state, dump_sa, (void *) pfk);
```

```
1554 int xfrm_state_walk(struct net *net, struct xfrm_state_walk *walk,
1555                     int (*func)(struct xfrm_state *, int, void*),
1556                     void *data)
1557 {
1558         struct xfrm_state *state;
1559         struct xfrm_state_walk *x;
1560         int err = 0;             
1561 
1562         if (walk->seq != 0 && list_empty(&walk->all))
1563                 return 0;
1564 
1565         spin_lock_bh(&xfrm_state_lock);
1566         if (list_empty(&walk->all)) <=== 根据walk->all 判定是全新的遍历，还是接着上次的位置往后遍历
1567                 x = list_first_entry(&net->xfrm.state_all, struct xfrm_state_walk, all);
1568         else
1569                 x = list_entry(&walk->all, struct xfrm_state_walk, all);
1570         list_for_each_entry_from(x, &net->xfrm.state_all, all) {
1571                 if (x->state == XFRM_STATE_DEAD)
1572                         continue;
1573                 state = container_of(x, struct xfrm_state, km);
1574                 if (!xfrm_id_proto_match(state->id.proto, walk->proto))
1575                         continue;
1576                 err = func(state, walk->seq, data); <=== 根据fun的返回决定是继续遍历下一个SA还是停止
1577                 if (err) {
1578                         list_move_tail(&walk->all, &x->all);
1579                         goto out;
1580                 }
1581                 walk->seq++;
1582         }
1583         if (walk->seq == 0) {
1584                 err = -ENOENT;
1585                 goto out;
1586         }
1587         list_del_init(&walk->all);
1588 out:
1589         spin_unlock_bh(&xfrm_state_lock);
1590         return err;
1591 }
1592 EXPORT_SYMBOL(xfrm_state_walk);
```

2. recvmsg
跟普通socket的recvmsg没有太大的区别。

调用skb_recv_datagram从socket的sk_receive_queue队列里摘一个skb下来， 并将skb的数据copy到用户空间。

检查SA是否已经全部dump了。 如果还有没被dump的sa，并且socket的有比较宽裕的空间，则再次激活SA dump。 
宽裕的剩余空间的检查，保证了dump的效率， 避免激活一次SA dump只能dump很少的SA。

```c
3609 static int pfkey_recvmsg(struct kiocb *kiocb,
3610                          struct socket *sock, struct msghdr *msg, size_t len,
3611                          int flags)
3612 {
3613         struct sock *sk = sock->sk;
3614         struct pfkey_sock *pfk = pfkey_sk(sk);
3615         struct sk_buff *skb;
3616         int copied, err;
3617 
3618         err = -EINVAL;
3619         if (flags & ~(MSG_PEEK|MSG_DONTWAIT|MSG_TRUNC|MSG_CMSG_COMPAT))
3620                 goto out;
3621 
3622         msg->msg_namelen = 0;
3623         skb = skb_recv_datagram(sk, flags, flags & MSG_DONTWAIT, &err);
3624         if (skb == NULL)
3625                 goto out;
3626 
3627         copied = skb->len;
3628         if (copied > len) {
3629                 msg->msg_flags |= MSG_TRUNC;
3630                 copied = len;
3631         }
3632 
3633         skb_reset_transport_header(skb);
3634         err = skb_copy_datagram_iovec(skb, 0, msg->msg_iov, copied);
3635         if (err)
3636                 goto out_free;
3637 
3638         sock_recv_ts_and_drops(msg, sk, skb);
3639 
3640         err = (flags & MSG_TRUNC) ? skb->len : copied;
3641 
3642         if (pfk->dump.dump != NULL &&
3643             3 * atomic_read(&sk->sk_rmem_alloc) <= sk->sk_rcvbuf)
3644                 pfkey_do_dump(pfk);
3645 
3646 out_free:
3647         skb_free_datagram(sk, skb);
3648 out:
3649         return err;
3650 }
```

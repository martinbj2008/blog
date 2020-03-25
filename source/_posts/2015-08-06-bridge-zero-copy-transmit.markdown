---
layout: post
title: "bridge zero copy transmit"
date: 2015-06-26 16:42:00 +0800
comments: true
categories: [virtnet]
tags: [kvm, vhost]
---

#### redhat 7 full support it
[How redhat 7 said](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Tuning_and_Optimization_Guide/sect-Virtualization_Tuning_Optimization_Guide-Networking-Techniques.html)
```
5.5.1. Bridge Zero Copy Transmit

Zero copy transmit mode is effective on large packet sizes. It typically reduces the host CPU overhead by up to 15% when transmitting large packets between a guest network and an external network, without affecting throughput.
It does not affect performance for guest-to-guest, guest-to-host, or small packet workloads.
Bridge zero copy transmit is fully supported on Red Hat Enterprise Linux 7 virtual machines, but disabled by default. To enable zero copy transmit mode, set the experimental_zcopytx kernel module parameter for the vhost_net module to 1.
NOTE
An additional data copy is normally created during transmit as a threat mitigation technique against denial of service and information leak attacks. Enabling zero copy transmit disables this threat mitigation technique.
If performance regression is observed, or if host CPU utilization is not a concern, zero copy transmit mode can be disabled by setting experimental_zcopytx to 0.
```

<!-- more -->

### related source
set flags
```
  handle_tx
    tun_sendmsg
      tun_get_user ** set destructor_arg as "struct ubuf_info *ubuf;", and set SKBTX_DEV_ZEROCOPY
```

### related source
set flags
callback

```
  __kfree_skb
    skb_release_all
      skb_release_data
        shinfo->destructor_arg ** "struct ubuf_info *ubuf;" and ubuf->callback is called.
```

```
 291 /* Expects to be always run from workqueue - which acts as
 292  * read-size critical section for our kind of RCU. */
 293 static void handle_tx(struct vhost_net *net)
 294 {
 295         struct vhost_net_virtqueue *nvq = &net->vqs[VHOST_NET_VQ_TX];
 296         struct vhost_virtqueue *vq = &nvq->vq;
....
 373                 /* use msg_control to pass vhost zerocopy ubuf info to skb */
 374                 if (zcopy_used) {
 375                         struct ubuf_info *ubuf;
 376                         ubuf = nvq->ubuf_info + nvq->upend_idx;
 377
 378                         vq->heads[nvq->upend_idx].id = cpu_to_vhost32(vq, head);
 379                         vq->heads[nvq->upend_idx].len = VHOST_DMA_IN_PROGRESS;
 380                         ubuf->callback = vhost_zerocopy_callback;
 381                         ubuf->ctx = nvq->ubufs;
 382                         ubuf->desc = nvq->upend_idx;
 383                         msg.msg_control = ubuf;
 384                         msg.msg_controllen = sizeof(ubuf);
 385                         ubufs = nvq->ubufs;
 386                         atomic_inc(&ubufs->refcount);
 387                         nvq->upend_idx = (nvq->upend_idx + 1) % UIO_MAXIOV;
 388                 } else {
 389                         msg.msg_control = NULL;
 390                         ubufs = NULL;
 391                 }
 392                 /* TODO: Check specific error and bomb out unless ENOBUFS? */
 393                 err = sock->ops->sendmsg(sock, &msg, len);
....
```

```
1502 static int tun_sendmsg(struct socket *sock, struct msghdr *m, size_t total_len)
1503 {
1504         int ret;
1505         struct tun_file *tfile = container_of(sock, struct tun_file, socket);
1506         struct tun_struct *tun = __tun_get(tfile);
1507
1508         if (!tun)
1509                 return -EBADFD;
1510
1511         ret = tun_get_user(tun, tfile, m->msg_control, &m->msg_iter,
1512                            m->msg_flags & MSG_DONTWAIT);
1513         tun_put(tun);
1514         return ret;
1515 }
```

```
1080 /* Get packet from user space buffer */
1081 static ssize_t tun_get_user(struct tun_struct *tun, struct tun_file *tfile,
1082                             void *msg_control, struct iov_iter *from,
1083                             int noblock)
1084 {
...
1249         /* copy skb_ubuf_info for callback when skb has no error */
1250         if (zerocopy) {
1251                 skb_shinfo(skb)->destructor_arg = msg_control;
1252                 skb_shinfo(skb)->tx_flags |= SKBTX_DEV_ZEROCOPY;
1253                 skb_shinfo(skb)->tx_flags |= SKBTX_SHARED_FRAG;
1254         }
...
```

```
 671 void __kfree_skb(struct sk_buff *skb)
 672 {
 673         skb_release_all(skb);
 674         kfree_skbmem(skb);
 675 }
 676 EXPORT_SYMBOL(__kfree_skb);
```

```
 654 /* Free everything but the sk_buff shell. */
 655 static void skb_release_all(struct sk_buff *skb)
 656 {
 657         skb_release_head_state(skb);
 658         if (likely(skb->head))
 659                 skb_release_data(skb);
 660 }
```


```
 572 static void skb_release_data(struct sk_buff *skb)
 573 {
 585         /*
 586          * If skb buf is from userspace, we need to notify the caller
 587          * the lower device DMA has done;
 588          */
 589         if (shinfo->tx_flags & SKBTX_DEV_ZEROCOPY) {
 590                 struct ubuf_info *uarg;
 591
 592                 uarg = shinfo->destructor_arg;
 593                 if (uarg->callback)
 594                         uarg->callback(uarg, true);
 595         }
 ...
601 }
```






### why it only affect performance for guest-to-nic? why guest-to-guest(host) not? 
<font color='red'>`skb_orphan_frags`</font>
#### The reason is network stack entry function will force clear `SKBTX_DEV_ZEROCOPY`
clear the flag and copy data from userspace.
```
2119 static inline int skb_orphan_frags(struct sk_buff *skb, gfp_t gfp_mask)
2120 {
2121         if (likely(!(skb_shinfo(skb)->tx_flags & SKBTX_DEV_ZEROCOPY)))
2122                 return 0;
2123         return skb_copy_ubufs(skb, gfp_mask);
2124 }
```

#### where `skb_orphan_frags` is used?
##### call trace
```
  __netif_receive_skb_core
    deliver_ptype_list_skb
      deliver_skb
        skb_orphan_frags
```

As it is known:  `__netif_receive_skb_core` is the entry of network stack.
```
3755 static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
3756 {
...
3871         type = skb->protocol;
3872
3873         /* deliver only exact match when indicated */
3874         if (likely(!deliver_exact)) {
3875                 deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
3876                                        &ptype_base[ntohs(type) &
3877                                                    PTYPE_HASH_MASK]);
3878         }
....
```

```
1772 static inline void deliver_ptype_list_skb(struct sk_buff *skb,
1773                                           struct packet_type **pt,
1774                                           struct net_device *orig_dev,
1775                                           __be16 type,
1776                                           struct list_head *ptype_list)
1777 {
1778         struct packet_type *ptype, *pt_prev = *pt;
1779
1780         list_for_each_entry_rcu(ptype, ptype_list, list) {
1781                 if (ptype->type != type)
1782                         continue;
1783                 if (pt_prev)
1784                         deliver_skb(skb, pt_prev, orig_dev);
1785                 pt_prev = ptype;
1786         }
1787         *pt = pt_prev;
1788 }
```

```
1762 static inline int deliver_skb(struct sk_buff *skb,
1763                               struct packet_type *pt_prev,
1764                               struct net_device *orig_dev)
1765 {
1766         if (unlikely(skb_orphan_frags(skb, GFP_ATOMIC)))
1767                 return -ENOMEM;
1768         atomic_inc(&skb->users);
1769         return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
1770 }
```

---
layout: post
title: "How does xfrm km work"
date: 2013-08-26 15:19
comments: true
categories: [xfrm]
tags: [xfrm, km]
---

### summary

There is a list `xfrm_km_list` in kernel.
Each node of the list is `struct xfrm_mgr`,
which has several methods to notify usersapce by netlink message.
Different methods has corresponding method,
and it broadcast the netlink message with different xfrm groups.


`struct xfrm_mgr` has many methods, for example:
	1. notify:  notify the sa change, ex: add, delete, expire ..
	2. acquire: notify when sp is match, while no SA is got.
	3. compile_policy:
	4. new_mapping:
	5: notify_policy: notify sp change. add, delete, expire.
	6. report:
	7. mirgrate:

<!-- more -->

#### `struct xfrm_mgr` and `xfrm_km_list`

```c
1630 static LIST_HEAD(xfrm_km_list);
```

```c
 579 struct xfrm_mgr {
 580         struct list_head        list;
 581         char                    *id;
 582         int                     (*notify)(struct xfrm_state *x, const struct km_event *c);
 583         int                     (*acquire)(struct xfrm_state *x, struct xfrm_tmpl *, struct xfrm_policy *xp);
 584         struct xfrm_policy      *(*compile_policy)(struct sock *sk, int opt, u8 *data, int len, int *dir);
 585         int                     (*new_mapping)(struct xfrm_state *x, xfrm_address_t *ipaddr, __be16 sport);
 586         int                     (*notify_policy)(struct xfrm_policy *x, int dir, const struct km_event *c);
 587         int                     (*report)(struct net *net, u8 proto, struct xfrm_selector *sel, xfrm_address_t *addr);
 588         int                     (*migrate)(const struct xfrm_selector *sel,
 589                                            u8 dir, u8 type,
 590                                            const struct xfrm_migrate *m,
 591                                            int num_bundles,
 592                                            const struct xfrm_kmaddress *k);
 593 };
```
#### `xfrm_register_km` vs `xfrm_unregister_km`

Register or unregister a `xfrm_mgr` to `xfrm_km_list`.

NOTE:  here we `use spin_lock_bh`,
because the `km_notify` method maybe used in irq bottom half.
ex: the first packet which trigger the dynamic sa
negotiation, which is processed in softirq(rx).

```c
1807 int xfrm_register_km(struct xfrm_mgr *km)
1808 {
1809         spin_lock_bh(&xfrm_km_lock);
1810         list_add_tail_rcu(&km->list, &xfrm_km_list);
1811         spin_unlock_bh(&xfrm_km_lock);
1812         return 0;
1813 }
```

```c
1816 int xfrm_unregister_km(struct xfrm_mgr *km)
1817 {
1818         spin_lock_bh(&xfrm_km_lock);
1819         list_del_rcu(&km->list);
1820         spin_unlock_bh(&xfrm_km_lock);
1821         synchronize_rcu();
1822         return 0;
1823 }
```

####  register with `netlink_mgr`
```c
3031 static int __init xfrm_user_init(void)
...
3040         rv = xfrm_register_km(&netlink_mgr);
...
```
```c
2989 static struct xfrm_mgr netlink_mgr = {
2990         .id             = "netlink",
2991         .notify         = xfrm_send_state_notify,
2992         .acquire        = xfrm_send_acquire,
2993         .compile_policy = xfrm_compile_policy,
2994         .notify_policy  = xfrm_send_policy_notify,
2995         .report         = xfrm_send_report,
2996         .migrate        = xfrm_send_migrate,
2997         .new_mapping    = xfrm_send_mapping,
2998 };
```

####  `km_state_expired`

```c
1643 void km_state_notify(struct xfrm_state *x, const struct km_event *c)
1644 {
1645         struct xfrm_mgr *km;
1646         rcu_read_lock();
1647         list_for_each_entry_rcu(km, &xfrm_km_list, list)
1648                 if (km->notify)
1649                         km->notify(x, c);
1650         rcu_read_unlock();
1651 }
```

```c
1656 void km_state_expired(struct xfrm_state *x, int hard, u32 portid)
1657 {
1658         struct net *net = xs_net(x);
1659         struct km_event c;
1660
1661         c.data.hard = hard;
1662         c.portid = portid;
1663         c.event = XFRM_MSG_EXPIRE;
1664         km_state_notify(x, &c);
1665
1666         if (hard)
1667                 wake_up(&net->xfrm.km_waitq);
1668 }
1669
1670 EXPORT_SYMBOL(km_state_expired);
```

#### `km_policy_expired` vs `km_policy_notify`
```c
1632 void km_policy_notify(struct xfrm_policy *xp, int dir, const struct km_event *c)
1633 {
1634         struct xfrm_mgr *km;
1635
1636         rcu_read_lock();
1637         list_for_each_entry_rcu(km, &xfrm_km_list, list)
1638                 if (km->notify_policy)
1639                         km->notify_policy(xp, dir, c);
1640         rcu_read_unlock();
1641 }
```

```c
1708 void km_policy_expired(struct xfrm_policy *pol, int dir, int hard, u32 portid)
1709 {
1710         struct net *net = xp_net(pol);
1711         struct km_event c;
1712
1713         c.data.hard = hard;
1714         c.portid = portid;
1715         c.event = XFRM_MSG_POLEXPIRE;
1716         km_policy_notify(pol, dir, &c);
1717
1718         if (hard)
1719                 wake_up(&net->xfrm.km_waitq);
1720 }
1721 EXPORT_SYMBOL(km_policy_expired);
```

#### `xfrm_send_state_notify`
```c
2577 static int xfrm_send_state_notify(struct xfrm_state *x, const struct km_event *c)
2578 {
2579
2580         switch (c->event) {
2581         case XFRM_MSG_EXPIRE:
2582                 return xfrm_exp_state_notify(x, c);
2583         case XFRM_MSG_NEWAE:
2584                 return xfrm_aevent_state_notify(x, c);
2585         case XFRM_MSG_DELSA:
2586         case XFRM_MSG_UPDSA:
2587         case XFRM_MSG_NEWSA:
2588                 return xfrm_notify_sa(x, c);
2589         case XFRM_MSG_FLUSHSA:
2590                 return xfrm_notify_sa_flush(c);
2591         default:
2592                 printk(KERN_NOTICE "xfrm_user: Unknown SA event %d\n",
2593                        c->event);
2594                 break;
2595         }
2596
2597         return 0;
2598
2599 }
```

```c
2428 static int xfrm_exp_state_notify(struct xfrm_state *x, const struct km_event *c)
2429 {
2430         struct net *net = xs_net(x);
2431         struct sk_buff *skb;
2432
2433         skb = nlmsg_new(xfrm_expire_msgsize(), GFP_ATOMIC);
2434         if (skb == NULL)
2435                 return -ENOMEM;
2436
2437         if (build_expire(skb, x, c) < 0) {
2438                 kfree_skb(skb);
2439                 return -EMSGSIZE;
2440         }
2441
2442         return nlmsg_multicast(net->xfrm.nlsk, skb, 0, XFRMNLGRP_EXPIRE, GFP_ATOMIC);
2443 }
```
#### `xfrm_send_acquire`
```c
2648 static int xfrm_send_acquire(struct xfrm_state *x, struct xfrm_tmpl *xt,
2649                              struct xfrm_policy *xp)
2650 {
2651         struct net *net = xs_net(x);
2652         struct sk_buff *skb;
2653
2654         skb = nlmsg_new(xfrm_acquire_msgsize(x, xp), GFP_ATOMIC);
2655         if (skb == NULL)
2656                 return -ENOMEM;
2657
2658         if (build_acquire(skb, x, xt, xp) < 0)
2659                 BUG();
2660
2661         return nlmsg_multicast(net->xfrm.nlsk, skb, 0, XFRMNLGRP_ACQUIRE, GFP_ATOMIC);
2662 }
```
#### how to notify strongswan negotiate a new sa.
```c
> xfrm_stat_find
> >  km_query
> > >  km->acquire
equal:
> > >  xfrm_send_acquire
```

```c
1671 /*
1672  * We send to all registered managers regardless of failure
1673  * We are happy with one success
1674 */
1675 int km_query(struct xfrm_state *x, struct xfrm_tmpl *t, struct xfrm_policy *pol)
1676 {
1677         int err = -EINVAL, acqret;
1678         struct xfrm_mgr *km;
1679
1680         rcu_read_lock();
1681         list_for_each_entry_rcu(km, &xfrm_km_list, list) {
1682                 acqret = km->acquire(x, t, pol);
1683                 if (!acqret)
1684                         err = acqret;
1685         }
1686         rcu_read_unlock();
1687         return err;
1688 }
1689 EXPORT_SYMBOL(km_query);
```

```c
 788 struct xfrm_state *
 789 xfrm_state_find(const xfrm_address_t *daddr, const xfrm_address_t *saddr,
 790                 const struct flowi *fl, struct xfrm_tmpl *tmpl,
 791                 struct xfrm_policy *pol, int *err,
 792                 unsigned short family)
 793 {
...
 838         x = best;
 839         if (!x && !error && !acquire_in_progress) {
 840                 if (tmpl->id.spi &&
 841                     (x0 = __xfrm_state_lookup(net, mark, daddr, tmpl->id.spi,
 842                                               tmpl->id.proto, encap_family)) != NULL) {
 843                         to_put = x0;
 844                         error = -EEXIST;
 845                         goto out;
 846                 }
 847                 x = xfrm_state_alloc(net);
 848                 if (x == NULL) {
 849                         error = -ENOMEM;
 850                         goto out;
 851                 }
 852                 /* Initialize temporary state matching only
 853                  * to current session. */
 854                 xfrm_init_tempstate(x, fl, tmpl, daddr, saddr, family);
 855                 memcpy(&x->mark, &pol->mark, sizeof(x->mark));
 856
 857                 error = security_xfrm_state_alloc_acquire(x, pol->security, fl->flowi_secid);
 858                 if (error) {
 859                         x->km.state = XFRM_STATE_DEAD;
 860                         to_put = x;
 861                         x = NULL;
 862                         goto out;
 863                 }
 864
 865                 if (km_query(x, tmpl, pol) == 0) {
 866                         x->km.state = XFRM_STATE_ACQ;
 867                         list_add(&x->km.all, &net->xfrm.state_all);
 868                         hlist_add_head(&x->bydst, net->xfrm.state_bydst+h);
```

#### pernet `net->xfrm.nlsk`
It is a basic of xfrm netlink.
All the message sent to/from kernel will use it.
See detail with blog about netlink.

http://martinbj2008.github.io/blog/2013/02/16/netlink-in-kernel
http://martinbj2008.github.io/blog/2013/03/15/netlink-in-kernel

```c
3000 static int __net_init xfrm_user_net_init(struct net *net)
3001 {
3002         struct sock *nlsk;
3003         struct netlink_kernel_cfg cfg = {
3004                 .groups = XFRMNLGRP_MAX,
3005                 .input  = xfrm_netlink_rcv,
3006         };
3007
3008         nlsk = netlink_kernel_create(net, NETLINK_XFRM, &cfg);
3009         if (nlsk == NULL)
3010                 return -ENOMEM;
3011         net->xfrm.nlsk_stash = nlsk; /* Don't set to NULL */
3012         rcu_assign_pointer(net->xfrm.nlsk, nlsk);
3013         return 0;
3014 }
```
#### `net->xfrm.km_waitq`

It is used to wait SA negotiation complete,
and is not related with km.

```c
2121         if (route == NULL && num_xfrms > 0) {
2122                 /* The only case when xfrm_bundle_lookup() returns a
2123                  * bundle with null route, is when the template could
2124                  * not be resolved. It means policies are there, but
2125                  * bundle could not be created, since we don't yet
2126                  * have the xfrm_state's. We need to wait for KM to
2127                  * negotiate new SA's or bail out with error.*/
2128                 if (net->xfrm.sysctl_larval_drop) {
2129                         /* EREMOTE tells the caller to generate
2130                          * a one-shot blackhole route. */
2131                         dst_release(dst);
2132                         xfrm_pols_put(pols, drop_pols);
2133                         XFRM_INC_STATS(net, LINUX_MIB_XFRMOUTNOSTATES);
2134
2135                         return make_blackhole(net, family, dst_orig);
2136                 }
2137                 if (fl->flowi_flags & FLOWI_FLAG_CAN_SLEEP) {
2138                         DECLARE_WAITQUEUE(wait, current);
2139
2140                         add_wait_queue(&net->xfrm.km_waitq, &wait);
2141                         set_current_state(TASK_INTERRUPTIBLE);
2142                         schedule();
2143                         set_current_state(TASK_RUNNING);
2144                         remove_wait_queue(&net->xfrm.km_waitq, &wait);
2145
2146                         if (!signal_pending(current)) {
2147                                 dst_release(dst);
2148                                 goto restart;
2149                         }
2150
2151                         err = -ERESTART;
2152                 } else
2153                         err = -EAGAIN;
2154
2155                 XFRM_INC_STATS(net, LINUX_MIB_XFRMOUTNOSTATES);
2156                 goto error;
2157         }
```

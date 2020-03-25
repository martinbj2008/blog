---
layout: post
title: "tcpdump work with bonding interface"
date: 2014-09-14 08:12
comments: true
categories: [netcore]
tag: TCPDUMP
---

### test case
####On redhat5, Why tcpdump could not work on bonding work.
OS: redhat 5.
There are two 82599 interfaces eth0 and eth1.
These two interfaces are used as slave of bond0,
eth1 is backup of eth0.

We ping the default gateway on test machine.
ping work OK, and tcpdump on bond0 show the icmp request and icmp require packets.
while on eth0 only icmp request, and eth1 has no any packet.

<!-- more -->

It is impossible there is no incoming packet on any physical interface.
Why tcpdump could not capture the packets on eth0.

### analysis
tcpdump is `pf_socket` which is based on `ptye_all`.
#### linux V2.6.32
In linux v2.6.32, there is bond process before `ptye_all`,
and thus the `skb->dev` will be change to `bond0` from `eth0`.
so when packet arrive `ptye_all`, ony match incoming dev `bond0`.
we has no chance to capture packet on physical interface eth0.

#### upstream linux v3.17-rc4
bond related process is moved to `dev->rx_handler`,
Just like the bridge or openvswitch.

Packet will first be processed by `ptype_all` with `skb->dev` is eth0
and then `rx_handler`(bond handler for eth0,eth1).
if the rx handler return `RX_HANDLER_ANOTHER`,
the packet arrive by `ptye_all` again with different`skb->dev` (bond0).

#### related patch
https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=5b2c4d
https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=63d8ea
https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=5b2c4dd

##### TODO:
Test with upstream kernel.

###Redhat source
redhat source is based on 2.6.32

```
2616 int __netif_receive_skb(struct sk_buff *skb)
2617 {
...
2636         if (!skb->iif)
2637                 skb->iif = skb->dev->ifindex;
2638 
2639         /*
2640          * bonding note: skbs received on inactive slaves should only
2641          * be delivered to pkt handlers that are exact matches.  Also
2642          * the deliver_no_wcard flag will be set.  If packet handlers
2643          * are sensitive to duplicate packets these skbs will need to
2644          * be dropped at the handler.  The vlan accel path may have
2645          * already set the deliver_no_wcard flag.
2646          */
2647 
2648         null_or_orig = NULL;
2649         orig_dev = skb->dev;
2650         if (skb->deliver_no_wcard)
2651                 null_or_orig = orig_dev;
2652         else if (orig_dev->master) {
2653                 if (skb_bond_should_drop(skb)) {
2654                         skb->deliver_no_wcard = 1;
2655                         null_or_orig = orig_dev; /* deliver only exact match */
2656                 } else
2657                         skb->dev = orig_dev->master;
2658         }
...
2677         list_for_each_entry_rcu(ptype, &ptype_all, list) {
2678                 if (ptype->dev == null_or_orig || ptype->dev == skb->dev ||
2679                     ptype->dev == orig_dev) {
2680                         if (pt_prev)
2681                                 ret = deliver_skb(skb, pt_prev, orig_dev);
2682                         pt_prev = ptype;
2683                 }
2684         }
```
#### upstream linux V3.16

```
3579 static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
3580 {
...
3604 another_round:
...
3626         list_for_each_entry_rcu(ptype, &ptype_all, list) {
3627                 if (!ptype->dev || ptype->dev == skb->dev) {
3628                         if (pt_prev)
3629                                 ret = deliver_skb(skb, pt_prev, orig_dev);
3630                         pt_prev = ptype;
3631                 }
3632         }
3633
3634 skip_taps:
3635 #ifdef CONFIG_NET_CLS_ACT
3636         skb = handle_ing(skb, &pt_prev, &ret, orig_dev);
3637         if (!skb)
3638                 goto unlock;
3639 ncls:
3640 #endif
3641
...
3656         rx_handler = rcu_dereference(skb->dev->rx_handler);
3657         if (rx_handler) {
3658                 if (pt_prev) {
3659                         ret = deliver_skb(skb, pt_prev, orig_dev);
3660                         pt_prev = NULL;
3661                 }
3662                 switch (rx_handler(&skb)) {
3663                 case RX_HANDLER_CONSUMED:
3664                         ret = NET_RX_SUCCESS;
3665                         goto unlock;
3666                 case RX_HANDLER_ANOTHER:
3667                         goto another_round;
3668                 case RX_HANDLER_EXACT:
3669                         deliver_exact = true;
3670                 case RX_HANDLER_PASS:
3671                         break;
3672                 default:
3673                         BUG();
3674                 }
3675         }
```

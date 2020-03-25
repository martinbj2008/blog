---
layout: post
title: "ipvlan study"
date: 2015-05-21 17:31:57 +0800
comments: true
categories: [route]
tags: [ipvlan]
---

#### L2 mode
##### xmit packet
###### xmit a normal pkt to other phy machine
```c
==> ipvlan_start_xmit
==> ==> ipvlan_xmit_mode_l2
==> ==> ==> skb->dev = ipvlan->phy_dev;
==> ==> ==> return dev_queue_xmit(skb);
```

######  xmit a normal pkt to other namespace
```c
==> ipvlan_start_xmit
==> ==> ipvlan_xmit_mode_l2
==> ==> ==> if (ether_addr_equal(eth->h_dest, eth->h_source))
==> ==> ==> addr = ipvlan_addr_lookup(ipvlan->port, lyr3h, addr_type, true);
==> ==> ==> ipvlan_rcv_frame(addr, skb, true);
==> ==> ==> ==> skb->dev = dev; <== dst namespace dev
==> ==> ==> ==> dev_forward_skb(ipvlan->dev, skb)
==> ==> ==> ==> ==> return __dev_forward_skb(dev, skb) ?: netif_rx_internal(skb);
==> ==> ==> ==> ==> ==> enqueue_to_backlog(skb, get_cpu(), &qtail);
```
###### xmit a mutlicast pkt
```c
==> ipvlan_start_xmit
==> ==> ipvlan_xmit_mode_l2
==> ==> ==> else if (is_multicast_ether_addr(eth->h_dest))
==> ==> ==> ipvlan_multicast_frame(ipvlan->port, skb, ipvlan, true);
==> ==> ==> ==> list_for_each_entry(ipvlan, &port->ipvlans, pnode)
==> ==> ==> ==> ==> dev_forward_skb or netif_rx(nskb);
```
##### recv packet 
All the packet are get by the `rx_handler`, `ipvlan_handle_frame`.

###### unicast packet
lookup the dest ipvlan port(net_device)
by the dst IPv4/6 address, and send to it.

```c
==> ipvlan_handle_frame
==> ==> ipvlan_handle_mode_l2
==> ==> ==> ipvlan_addr_lookup(port, lyr3h, addr_type, true);
==>  ==> ==> ==> skb->dev = dev;
==> ==> ==> ==> dev_forward_skb or ret = RX_HANDLER_ANOTHER;
```
###### multicast packet.
```c
==> ipvlan_handle_frame
==> ==> ipvlan_handle_mode_l2
==> ==> ==> if (is_multicast_ether_addr(eth->h_dest)) {
==> ==> ==> ipvlan_addr_lookup(port, lyr3h, addr_type, true);
==> ==> ==> ==> if (ipvlan_external_frame(skb, port))
==> ==>  ==> ==> ==> ipvlan_multicast_frame(port, skb, NULL, false);
```

#### l3 mode
```c
ipvlan_start_xmit
```

```c
308 static const struct net_device_ops ipvlan_netdev_ops = {
309         .ndo_init               = ipvlan_init,
310         .ndo_uninit             = ipvlan_uninit,
311         .ndo_open               = ipvlan_open,
312         .ndo_stop               = ipvlan_stop,
313         .ndo_start_xmit         = ipvlan_start_xmit,
314         .ndo_fix_features       = ipvlan_fix_features,
315         .ndo_change_rx_flags    = ipvlan_change_rx_flags,
316         .ndo_set_rx_mode        = ipvlan_set_multicast_mac_filter,
317         .ndo_get_stats64        = ipvlan_get_stats64,
318         .ndo_vlan_rx_add_vid    = ipvlan_vlan_rx_add_vid,
319         .ndo_vlan_rx_kill_vid   = ipvlan_vlan_rx_kill_vid,
320 };
```

```c
495 int ipvlan_queue_xmit(struct sk_buff *skb, struct net_device *dev)
496 {
497         struct ipvl_dev *ipvlan = netdev_priv(dev);
498         struct ipvl_port *port = ipvlan_port_get_rcu(ipvlan->phy_dev);
499
500         if (!port)
501                 goto out;
502
503         if (unlikely(!pskb_may_pull(skb, sizeof(struct ethhdr))))
504                 goto out;
505
506         switch(port->mode) {
507         case IPVLAN_MODE_L2:
508                 return ipvlan_xmit_mode_l2(skb, dev);
509         case IPVLAN_MODE_L3:
510                 return ipvlan_xmit_mode_l3(skb, dev);
511         }
512
513         /* Should not reach here */
514         WARN_ONCE(true, "ipvlan_queue_xmit() called for mode = [%hx]\n",
515                           port->mode);
516 out:
517         kfree_skb(skb);
518         return NET_XMIT_DROP;
519 }
```

```c
457 static int ipvlan_xmit_mode_l2(struct sk_buff *skb, struct net_device *dev)
458 {
459         const struct ipvl_dev *ipvlan = netdev_priv(dev);
460         struct ethhdr *eth = eth_hdr(skb);
461         struct ipvl_addr *addr;
462         void *lyr3h;
463         int addr_type;
464
465         if (ether_addr_equal(eth->h_dest, eth->h_source)) {
466                 lyr3h = ipvlan_get_L3_hdr(skb, &addr_type);
467                 if (lyr3h) {
468                         addr = ipvlan_addr_lookup(ipvlan->port, lyr3h, addr_type, true);
469                         if (addr)
470                                 return ipvlan_rcv_frame(addr, skb, true);
471                 }
472                 skb = skb_share_check(skb, GFP_ATOMIC);
473                 if (!skb)
474                         return NET_XMIT_DROP;
475
476                 /* Packet definitely does not belong to any of the
477                  * virtual devices, but the dest is local. So forward
478                  * the skb for the main-dev. At the RX side we just return
479                  * RX_PASS for it to be processed further on the stack.
480                  */
481                 return dev_forward_skb(ipvlan->phy_dev, skb);
482
483         } else if (is_multicast_ether_addr(eth->h_dest)) {
484                 u8 ip_summed = skb->ip_summed;
485
486                 skb->ip_summed = CHECKSUM_UNNECESSARY;
487                 ipvlan_multicast_frame(ipvlan->port, skb, ipvlan, true);
488                 skb->ip_summed = ip_summed;
489         }
490
491         skb->dev = ipvlan->phy_dev;
492         return dev_queue_xmit(skb);
493 }
```

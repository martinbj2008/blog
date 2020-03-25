---
layout: post
title: "xen tcp checksum for back and front net driver"
date: 2015-06-26 16:42:00 +0800
comments: true
categories: [virtnet]
tags: [checksum]
---

#### Test case
two linux guest vm on a same host(physical machine), there is a tcp session
between them.

#### conclusion:
the checksum in tcp header is only cover the faked tcp header not the whole
tcp packet.
##### call trace
vm1(netfront) ---> host backend(vif_1) ---> bridge ---> host backend(vif_2) ---> vm2(netfrontend)

<!-- more -->

###### netfront
before sent to backend, netfront driver set two flags:
`XEN_NETTXF_csum_blank | XEN_NETTXF_data_validated;"`
```
 575         if (skb->ip_summed == CHECKSUM_PARTIAL)
 576                 /* local packet? */
 577                 tx->flags |= XEN_NETTXF_csum_blank | XEN_NETTXF_data_validated;
 578         else if (skb->ip_summed == CHECKSUM_UNNECESSARY)
 579                 /* remote but checksummed. */
 580                 tx->flags |= XEN_NETTXF_data_validated;
```
###### net backend
```
1409 static int xenvif_tx_submit(struct xenvif_queue *queue)
1410 {
...
1452                 if (txp->flags & XEN_NETTXF_csum_blank)
1453                         skb->ip_summed = CHECKSUM_PARTIAL;
1454                 else if (txp->flags & XEN_NETTXF_data_validated)
1455                         skb->ip_summed = CHECKSUM_UNNECESSARY;
...
1515                 netif_receive_skb(skb);
```

###### bridge
forward this packet to vm2's backend.

###### netbackend
after bridge, skb reach peer vm2's backend's ndo xmit function.

```
139 static int xenvif_start_xmit(struct sk_buff *skb, struct net_device *dev)

...
497 static void xenvif_rx_action(struct xenvif_queue *queue)
498 {
...
 567
 568                 if (skb->ip_summed == CHECKSUM_PARTIAL) /* local packet? */
 569                         flags |= XEN_NETRXF_csum_blank | XEN_NETRXF_data_validated;
 570                 else if (skb->ip_summed == CHECKSUM_UNNECESSARY)
 571                         /* remote but checksummed. */
 572                         flags |= XEN_NETRXF_data_validated;
```
###### netfront
xen net front driver will do nothing. just setup the csum offset and len in skb,
even the fake tcp header checksum is not re-calculated.
```
 876 static int handle_incoming_queue(struct netfront_queue *queue,
 877                                  struct sk_buff_head *rxq)
 878 {
 ...
 893                 if (checksum_setup(queue->info->netdev, skb)) {
 894                         kfree_skb(skb);
 895                         packets_dropped++;
 896                         queue->info->netdev->stats.rx_errors++;
 897                         continue;
 898                 }
 ...
 905                 /* Pass it up. */
 906                 napi_gro_receive(&queue->napi, skb);
 907         }
```

```
 852 static int checksum_setup(struct net_device *dev, struct sk_buff *skb)
 853 {
 854         bool recalculate_partial_csum = false;
 855
 856         /*
 857          * A GSO SKB must be CHECKSUM_PARTIAL. However some buggy
 858          * peers can fail to set NETRXF_csum_blank when sending a GSO
 859          * frame. In this case force the SKB to CHECKSUM_PARTIAL and
 860          * recalculate the partial checksum.
 861          */
 862         if (skb->ip_summed != CHECKSUM_PARTIAL && skb_is_gso(skb)) {
 863                 struct netfront_info *np = netdev_priv(dev);
 864                 atomic_inc(&np->rx_gso_checksum_fixup);
 865                 skb->ip_summed = CHECKSUM_PARTIAL;
 866                 recalculate_partial_csum = true;
 867         }
 868
 869         /* A non-CHECKSUM_PARTIAL SKB does not require setup. */
 870         if (skb->ip_summed != CHECKSUM_PARTIAL)
 871                 return 0;
 872
 873         return skb_checksum_setup(skb, recalculate_partial_csum);
 874 }
```

```
4006 /**
4007  * skb_checksum_setup - set up partial checksum offset
4008  * @skb: the skb to set up
4009  * @recalculate: if true the pseudo-header checksum will be recalculated
4010  */
4011 int skb_checksum_setup(struct sk_buff *skb, bool recalculate)
4012 {
4013         int err;
4014
4015         switch (skb->protocol) {
4016         case htons(ETH_P_IP):
4017                 err = skb_checksum_setup_ipv4(skb, recalculate);
4018                 break;
4019
4020         case htons(ETH_P_IPV6):
4021                 err = skb_checksum_setup_ipv6(skb, recalculate);
4022                 break;
4023
4024         default:
4025                 err = -EPROTO;
4026                 break;
4027         }
4028
4029         return err;
4030 }
4031 EXPORT_SYMBOL(skb_checksum_setup);
```

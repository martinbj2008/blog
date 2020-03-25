---
layout: post
title: "how does ixgbe use queue index"
date: 2014-09-16 10:58
comments: true
categories:
---

### data structure
```
7913 static const struct net_device_ops ixgbe_netdev_ops = {
...
7916         .ndo_start_xmit         = ixgbe_xmit_frame,
7917         .ndo_select_queue       = ixgbe_select_queue,
...
```
### receive skb: record receive queue index

record `queue_index +1`, 0 is used as NOT record.

####call trace
```
> ixgbe_poll
> > ixgbe_clean_rx_irq
> > > ixgbe_process_skb_fields
> > > > skb_record_rx_queue
```

<!-- more -->
```
1675 static void ixgbe_process_skb_fields(struct ixgbe_ring *rx_ring,
1676                                      union ixgbe_adv_rx_desc *rx_desc,
1677                                      struct sk_buff *skb)
1678 {
1679         struct net_device *dev = rx_ring->netdev;
1680
1681         ixgbe_update_rsc_stats(rx_ring, skb);
1682
1683         ixgbe_rx_hash(rx_ring, rx_desc, skb);
1684
1685         ixgbe_rx_checksum(rx_ring, rx_desc, skb);
1686
1687         if (unlikely(ixgbe_test_staterr(rx_desc, IXGBE_RXDADV_STAT_TS)))
1688                 ixgbe_ptp_rx_hwtstamp(rx_ring->q_vector->adapter, skb);
1689
1690         if ((dev->features & NETIF_F_HW_VLAN_CTAG_RX) &&
1691             ixgbe_test_staterr(rx_desc, IXGBE_RXD_STAT_VP)) {
1692                 u16 vid = le16_to_cpu(rx_desc->wb.upper.vlan);
1693                 __vlan_hwaccel_put_tag(skb, htons(ETH_P_8021Q), vid);
1694         }
1695
1696         skb_record_rx_queue(skb, rx_ring->queue_index);
1697
1698         skb->protocol = eth_type_trans(skb, dev);
1699 }
```

```
3004 static inline void skb_record_rx_queue(struct sk_buff *skb, u16 rx_queue)
3005 {
3006         skb->queue_mapping = rx_queue + 1;
3007 }
```

### `dev_queue_xmit` select queue
KEY: ixgbe driver will first check if queue index is recorded(`skb->queue_mapping != 0`),
if record, return turn queue index value(`skb->queue_mapping -1`).
the queue index value is stored again to `skb->queue_mapping` by `skb_set_queue_mapping` in function `netdev_pick_tx`.

#### call trace
```
> dev_queue_xmit
> > __dev_queue_xmit
> > > netdev_pick_tx
> > > > ops->ndo_select_queue
> > > > ixgbe_select_queue
> > > > > skb_rx_queue_recorded(skb) ? skb_get_rx_queue(skb) : smp_processor_id();
> > > > skb_set_queue_mapping
```

```
398 struct netdev_queue *netdev_pick_tx(struct net_device *dev,
399                                     struct sk_buff *skb,
400                                     void *accel_priv)
401 {
402         int queue_index = 0;
403
404         if (dev->real_num_tx_queues != 1) {
405                 const struct net_device_ops *ops = dev->netdev_ops;
406                 if (ops->ndo_select_queue)
407                         queue_index = ops->ndo_select_queue(dev, skb, accel_priv,
408                                                             __netdev_pick_tx);
409                 else
410                         queue_index = __netdev_pick_tx(dev, skb);
411
412                 if (!accel_priv)
413                         queue_index = netdev_cap_txqueue(dev, queue_index);
414         }
415
416         skb_set_queue_mapping(skb, queue_index);
417         return netdev_get_tx_queue(dev, queue_index);
418 }
```
```
7096 static u16 ixgbe_select_queue(struct net_device *dev, struct sk_buff *skb,
7097                               void *accel_priv, select_queue_fallback_t fallback)
7098 {
7099         struct ixgbe_fwd_adapter *fwd_adapter = accel_priv;
7100 #ifdef IXGBE_FCOE
7101         struct ixgbe_adapter *adapter;
7102         struct ixgbe_ring_feature *f;
7103         int txq;
7104 #endif
7105
7106         if (fwd_adapter)
7107                 return skb->queue_mapping + fwd_adapter->tx_base_queue;
7108
7109 #ifdef IXGBE_FCOE
7110
7111         /*
7112          * only execute the code below if protocol is FCoE
7113          * or FIP and we have FCoE enabled on the adapter
7114          */
7115         switch (vlan_get_protocol(skb)) {
7116         case htons(ETH_P_FCOE):
7117         case htons(ETH_P_FIP):
7118                 adapter = netdev_priv(dev);
7119
7120                 if (adapter->flags & IXGBE_FLAG_FCOE_ENABLED)
7121                         break;
7122         default:
7123                 return fallback(dev, skb);
7124         }
7125
7126         f = &adapter->ring_feature[RING_F_FCOE];
7127
7128         txq = skb_rx_queue_recorded(skb) ? skb_get_rx_queue(skb) :
7129                                            smp_processor_id();
7130
7131         while (txq >= f->indices)
7132                 txq -= f->indices;
7133
7134         return txq + f->offset;
7135 #else
7136         return fallback(dev, skb);
7137 #endif
7138 }
```
```
2989 static inline void skb_set_queue_mapping(struct sk_buff *skb, u16 queue_mapping)
2990 {
2991         skb->queue_mapping = queue_mapping;
2992 }
```

```
3014 static inline bool skb_rx_queue_recorded(const struct sk_buff *skb)
3015 {
3016         return skb->queue_mapping != 0;
3017 }
```
### xmit skb use selected queue
use the corresponding queue by the queue index `skb->queue_mapping` which is based on 0.

```
7298 static netdev_tx_t ixgbe_xmit_frame(struct sk_buff *skb,
7299                                     struct net_device *netdev)
7300 {
7301         return __ixgbe_xmit_frame(skb, netdev, NULL);
7302 }
```

```
7275 static netdev_tx_t __ixgbe_xmit_frame(struct sk_buff *skb,
7276                                       struct net_device *netdev,
7277                                       struct ixgbe_ring *ring)
7278 {
7279         struct ixgbe_adapter *adapter = netdev_priv(netdev);
7280         struct ixgbe_ring *tx_ring;
7281 
7282         /*
7283          * The minimum packet size for olinfo paylen is 17 so pad the skb
7284          * in order to meet this minimum size requirement.
7285          */
7286         if (unlikely(skb->len < 17)) {
7287                 if (skb_pad(skb, 17 - skb->len))
7288                         return NETDEV_TX_OK;
7289                 skb->len = 17;
7290                 skb_set_tail_pointer(skb, 17);
7291         }
7292 
7293         tx_ring = ring ? ring : adapter->tx_ring[skb->queue_mapping];
7294 
7295         return ixgbe_xmit_frame_ring(skb, adapter, tx_ring);
7296 }
```


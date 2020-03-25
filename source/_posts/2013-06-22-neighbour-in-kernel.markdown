---
layout: post
title: "neighbour 学习笔记（kernel 3.0)"
date: 2011-09-28 00:00
comments: true
categories: [netcore]
tags: [neighbour, kernel, TODO]
---

<!-- more -->

1. For ethernet, dev->header_ops is eth_header_ops
```
 936 static int __devinit e1000_probe(struct pci_dev *pdev,          
 937                                  const struct pci_device_id *ent)
 938  
...
 973         netdev = alloc_etherdev(sizeof(struct e1000_adapter));
```

include/linux/etherdevice.h
------------------------------------------
```
 53 #define alloc_etherdev(sizeof_priv) alloc_etherdev_mq(sizeof_priv, 1)
 54 #define alloc_etherdev_mq(sizeof_priv, count) alloc_etherdev_mqs(sizeof_priv, count, count)
```

net/ethernet/eth.c
---------------------
```
365 struct net_device *alloc_etherdev_mqs(int sizeof_priv, unsigned int txqs,
366                                       unsigned int rxqs)
367 {
368         return alloc_netdev_mqs(sizeof_priv, "eth%d", ether_setup, txqs, rxqs);
369 }
```

net/core/dev.c
---------------------
```
5821 struct net_device *alloc_netdev_mqs(int sizeof_priv, const char *name,
5822                 void (*setup)(struct net_device *), 
5823                 unsigned int txqs, unsigned int rxqs)
...
5880         dev->priv_flags = IFF_XMIT_DST_RELEASE;
5881         setup(dev); <=== 
5882 
5883         dev->num_tx_queues = txqs;
```

net/ethernet/eth.c
---------------------
```
334 void ether_setup(struct net_device *dev)
336         dev->header_ops         = &eth_header_ops;<===
337         dev->type               = ARPHRD_ETHER;
338         dev->hard_header_len    = ETH_HLEN;
339         dev->mtu                = ETH_DATA_LEN;
340         dev->addr_len           = ETH_ALEN;
341         dev->tx_queue_len       = 1000; /* Ethernet wants good queues */
342         dev->flags              = IFF_BROADCAST|IFF_MULTICAST;
343         dev->priv_flags         = IFF_TX_SKB_SHARING;
344 
345         memset(dev->broadcast, 0xFF, ETH_ALEN);
346 
347 }
```

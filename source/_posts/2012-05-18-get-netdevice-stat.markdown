---
layout: post
title: "Get netdevice stat"
date: 2012-05-18 00:00
comments: true
categories: [netcore]
tags: [kernel, netdev]
---

##call trace

the ‘.show’ method in device_attribute will call the netstat_show,
some of the driver read the part of stat from NIC register,
but most count the stat by software in `dev->stat`
```c
 > dev_get_stats
 > > dev_get_stats
 > > > ops = dev->netdev_ops;
 > > > > ops->ndo_get_stats64
 > > > > > ixgbe_get_stats64,
```
<!-- more -->

```c
356 #define NETSTAT_ENTRY(name)                                             \
357 static ssize_t show_##name(struct device *d,                            \
358                            struct device_attribute *attr, char *buf)    \
359 {                                                                       \
360         return netstat_show(d, attr, buf,                               \
361                             offsetof(struct rtnl_link_stats64, name));  \
362 }
333 /* Show a given an attribute in the statistics group */
334 static ssize_t netstat_show(const struct device *d,
335                             struct device_attribute *attr, char *buf,
336                             unsigned long offset)
337 {
338         struct net_device *dev = to_net_dev(d);
339         ssize_t ret = -EINVAL;
340
341         WARN_ON(offset > sizeof(struct rtnl_link_stats64) ||
342                         offset % sizeof(u64) != 0);
343
344         read_lock(&dev_base_lock);
345         if (dev_isalive(dev)) {
346                 struct rtnl_link_stats64 temp;
347                 const struct rtnl_link_stats64 *stats = dev_get_stats(dev, &temp); <====
348
349                 ret = sprintf(buf, fmt_u64, *(u64 *)(((u8 *) stats) + offset));
350         }
351         read_unlock(&dev_base_lock);
352         return ret;
353 }
```

```c
5855 struct rtnl_link_stats64 *dev_get_stats(struct net_device *dev,
5856                                         struct rtnl_link_stats64 *storage)
5857 {
5858         const struct net_device_ops *ops = dev->netdev_ops;
5859
5860         if (ops->ndo_get_stats64) {
5861                 memset(storage, 0, sizeof(*storage));
5862                 ops->ndo_get_stats64(dev, storage); <== for IXGBE netcard driver.
5863         } else if (ops->ndo_get_stats) {
5864                 netdev_stats_to_stats64(storage, ops->ndo_get_stats(dev)); <=== E1000 driver.
5865         } else {
5866                 netdev_stats_to_stats64(storage, &dev->stats);
5867         }
5868         storage->rx_dropped += atomic_long_read(&dev->rx_dropped);
5869         return storage;
5870 }
5871 EXPORT_SYMBOL(dev_get_stats);
```

```c
6684 static const struct net_device_ops ixgbe_netdev_ops = {
...
6702         .ndo_get_stats64        = ixgbe_get_stats64,
1
2
6450 static struct rtnl_link_stats64 *ixgbe_get_stats64(struct net_device *netdev,
6451                                                    struct rtnl_link_stats64 *stats)
```

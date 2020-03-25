---
layout: post
title: "callback notify when add a IPv4 add"
date: 2013-08-20 17:15
comments: true
categories: [netcore]
tags: [IPv4, route, address]
---

##summary
IP4 route register a call back notify in IPv4 address.
When a address is added, `fib_inetaddr_event` will be called.

and then 4 route entry maybe added in `fib_add_ifaddr`

<!-- more -->



```c
1273 int register_inetaddr_notifier(struct notifier_block *nb)
1274 { 
1275         return blocking_notifier_chain_register(&inetaddr_chain, nb);
1276 } 
1277 EXPORT_SYMBOL(register_inetaddr_notifier)
```

```c
1075 static struct notifier_block fib_inetaddr_notifier = { 
1076         .notifier_call = fib_inetaddr_event,     
1077 };
```

```c
1008 static int fib_inetaddr_event(struct notifier_block *this, unsigned long event, void *ptr)
1009 {       
1010         struct in_ifaddr *ifa = (struct in_ifaddr *)ptr;
1011         struct net_device *dev = ifa->ifa_dev->dev;
1012         struct net *net = dev_net(dev);
1013         
1014         switch (event) {
1015         case NETDEV_UP:
1016                 fib_add_ifaddr(ifa);
1017 #ifdef CONFIG_IP_ROUTE_MULTIPATH
1018                 fib_sync_up(dev);
1019 #endif  
1020                 atomic_inc(&net->ipv4.dev_addr_genid);
1021                 rt_cache_flush(dev_net(dev));
1022                 break;
1023         case NETDEV_DOWN:
1024                 fib_del_ifaddr(ifa, NULL);
1025                 atomic_inc(&net->ipv4.dev_addr_genid);
1026                 if (ifa->ifa_dev->ifa_list == NULL) {
1027                         /* Last address was deleted from this interface.
1028                          * Disable IP.
1029                          */
1030                         fib_disable_ip(dev, 1);
1031                 } else {
1032                         rt_cache_flush(dev_net(dev));
1033                 }
1034                 break;
1035         }
1036         return NOTIFY_DONE;
1037 }
```

```c
 734 void fib_add_ifaddr(struct in_ifaddr *ifa)
 735 {               
 736         struct in_device *in_dev = ifa->ifa_dev;
 737         struct net_device *dev = in_dev->dev;
 738         struct in_ifaddr *prim = ifa;
 739         __be32 mask = ifa->ifa_mask;
 740         __be32 addr = ifa->ifa_local;
 741         __be32 prefix = ifa->ifa_address & mask;
 742                 
 743         if (ifa->ifa_flags & IFA_F_SECONDARY) {
 744                 prim = inet_ifa_byprefix(in_dev, prefix, mask);
 745                 if (prim == NULL) {
 746                         pr_warn("%s: bug: prim == NULL\n", __func__);
 747                         return;
 748                 } 
 749         }               
 750                 
 751         fib_magic(RTM_NEWROUTE, RTN_LOCAL, addr, 32, prim);
 752         
 753         if (!(dev->flags & IFF_UP))
 754                 return;
 755 
 756         /* Add broadcast address, if it is explicitly assigned. */
 757         if (ifa->ifa_broadcast && ifa->ifa_broadcast != htonl(0xFFFFFFFF))
 758                 fib_magic(RTM_NEWROUTE, RTN_BROADCAST, ifa->ifa_broadcast, 32, prim);
 759         
 760         if (!ipv4_is_zeronet(prefix) && !(ifa->ifa_flags & IFA_F_SECONDARY) &&
 761             (prefix != addr || ifa->ifa_prefixlen < 32)) {
 762                 fib_magic(RTM_NEWROUTE,
 763                           dev->flags & IFF_LOOPBACK ? RTN_LOCAL : RTN_UNICAST,
 764                           prefix, ifa->ifa_prefixlen, prim);
 765 
 766                 /* Add network specific broadcasts, when it takes a sense */
 767                 if (ifa->ifa_prefixlen < 31) {
 768                         fib_magic(RTM_NEWROUTE, RTN_BROADCAST, prefix, 32, prim);
 769                         fib_magic(RTM_NEWROUTE, RTN_BROADCAST, prefix | ~mask,
 770                                   32, prim);
 771                 }
 772         }
 773 }
```

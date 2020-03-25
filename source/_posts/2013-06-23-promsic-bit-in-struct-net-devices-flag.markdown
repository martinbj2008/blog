---
layout: post
title: "PROMISC in net\_device->flag"
date: 2013-06-23 06:01
comments: true
categories: [netlink]
tags: [kernel, netlink]
---
summary
====

promisc is one bit of struct net\_device's flag,  which is used to indicate if a device is in promisc status.
``` c
30 /* Standard interface flags (netdevice->flags). */
31 #define IFF_UP          0x1             /* interface is up              */
32 #define IFF_BROADCAST   0x2             /* broadcast address valid      */
33 #define IFF_DEBUG       0x4             /* turn on debugging            */
34 #define IFF_LOOPBACK    0x8             /* is a loopback net            */
35 #define IFF_POINTOPOINT 0x10            /* interface is has p-p link    */
36 #define IFF_NOTRAILERS  0x20            /* avoid use of trailers        */
37 #define IFF_RUNNING     0x40            /* interface RFC2863 OPER_UP    */
38 #define IFF_NOARP       0x80            /* no ARP protocol              */
39 #define IFF_PROMISC     0x100           /* receive all packets          */
40 #define IFF_ALLMULTI    0x200           /* receive all multicast packets*/
...
```
There are two kinds of operataion, could cause a NIC enter/leave promisc status.

1. ip command    
	run mutli `on` command, just need one `off` to recover.
```sh
   	ip link set dev eth0 promisc on
	ip link set dev eth0 promisc off
```

2. tcpdump command    
When tcpdump starts, it let dev to promisc,
and just before exit, tcpdump let dev left promisc.
All these is done by call kernel api dev_set_promiscuity.

<!-- more -->

Data struct and function
------------------------------
summary
------------------------------
kernel 3.10 rc6

There is a element `promiscuity` in `struct net device`,
which is reference for NIC promisc status.
Every time we want to set promisc to NIC, the reference increase 1,
while unset promisc, the value sub 1.

```c
1156         unsigned int            promiscuity;
```
When the value change from 0 to 1, or become 0,
that netdevice's ops is called to set/unset promisic status.

call trace
---------------------
```c
> dev_set_promiscuity    
>> __dev_set_promiscuity    
>>> dev->promiscuity += inc;    
>>>> dev->flags |= IFF_PROMISC or  &= ~IFF_PROMISC    
>>> dev_change_rx_flags(dev, IFF_PROMISC);    
>>>> ops->ndo_change_rx_flags(dev, flags);<== for most nic, the method is null.        
>> dev_set_rx_mode    
>>> const struct net_device_ops *ops = dev->netdev_ops;    
>>> ops->ndo_set_rx_mode(dev);
>>> for e100, e100_set_multicast_list 
```

dev_set_promiscuity
---
call __dev_set_promiscuity to record the reference of promisc,
If real need change status(refer from 0 to 1, or from 1 to 0),
call dev_set_rx_mode to really change NIC promisc status.

```c
4490 /**
4491  *      dev_set_promiscuity     - update promiscuity count on a device
4492  *      @dev: device
4493  *      @inc: modifier
4494  *
4495  *      Add or remove promiscuity from a device. While the count in the device
4496  *      remains above zero the interface remains promiscuous. Once it hits zero
4497  *      the device reverts back to normal filtering operation. A negative inc
4498  *      value is used to drop promiscuity on the device.
4499  *      Return 0 if successful or a negative errno code on error.
4500  */
4501 int dev_set_promiscuity(struct net_device *dev, int inc)
4502 {
4503         unsigned int old_flags = dev->flags;
4504         int err;
4505
4506         err = __dev_set_promiscuity(dev, inc);
4507         if (err < 0)
4508                 return err;
4509         if (dev->flags != old_flags)
4510                 dev_set_rx_mode(dev);
4511         return err;
4512 }
4513 EXPORT_SYMBOL(dev_set_promiscuity);
```

\_\_dev_set_promiscuity
----
set the
```c
static int __dev_set_promiscuity(struct net_device *dev, int inc)
4445 {
4446         unsigned int old_flags = dev->flags;
...
4452         dev->flags |= IFF_PROMISC;
4453         dev->promiscuity += inc;
4454         if (dev->promiscuity == 0) {
...
4460                         dev->flags &= ~IFF_PROMISC;
...
4467         }
4468         if (dev->flags != old_flags) {
...
4485                 dev_change_rx_flags(dev, IFF_PROMISC);
4486         }
```

```c
4436 static void dev_change_rx_flags(struct net_device *dev, int flags)
4437 {
4438         const struct net_device_ops *ops = dev->netdev_ops;
4439/*Junwei most nic has no this ops ndo_change_rx_flags*/
4440         if ((dev->flags & IFF_UP) && ops->ndo_change_rx_flags)
4441                 ops->ndo_change_rx_flags(dev, flags);
4442 }
```

```c
4592 void dev_set_rx_mode(struct net_device *dev)
4593 {
4594         netif_addr_lock_bh(dev);
4595         __dev_set_rx_mode(dev);
4596         netif_addr_unlock_bh(dev);
4597 }
```

```c
4558 /*
4559  *      Upload unicast and multicast address lists to device and
4560  *      configure RX filtering. When the device doesn't support unicast
4561  *      filtering it is put in promiscuous mode while unicast addresses
4562  *      are present.
4563  */
4564 void __dev_set_rx_mode(struct net_device *dev)
4565 {
4566         const struct net_device_ops *ops = dev->netdev_ops;
4567
4568         /* dev_open will call this function so the list will stay sane. */
4569         if (!(dev->flags&IFF_UP))
4570                 return;
4571
4572         if (!netif_device_present(dev))
4573                 return;
4574
4575         if (!(dev->priv_flags & IFF_UNICAST_FLT)) {
4576                 /* Unicast addresses changes may only happen under the rtnl,
4577                  * therefore calling __dev_set_promiscuity here is safe.
4578                  */
4579                 if (!netdev_uc_empty(dev) && !dev->uc_promisc) {
4580                         __dev_set_promiscuity(dev, 1);
4581                         dev->uc_promisc = true;
4582                 } else if (netdev_uc_empty(dev) && dev->uc_promisc) {
4583                         __dev_set_promiscuity(dev, -1);
4584                         dev->uc_promisc = false;
4585                 }
4586         }
4587
4588         if (ops->ndo_set_rx_mode)
4589                 ops->ndo_set_rx_mode(dev); <===
4590 }
```
for example e100 nic, ops->ndo_set_rx_mode is in file
drivers/net/ethernet/intel/e100.c

```c
2830 static const struct net_device_ops e100_netdev_ops = {
2835         .ndo_set_rx_mode        = e100_set_multicast_list, <===
```
```c
1609 static void e100_set_multicast_list(struct net_device *netdev)
1610 {
...
1617         if (netdev->flags & IFF_PROMISC)
1618                 nic->flags |= promiscuous;
1619         else
1620                 nic->flags &= ~promiscuous;
...
1628         e100_exec_cb(nic, NULL, e100_configure);
1629         e100_exec_cb(nic, NULL, e100_multi);
1630 }
```
how tcpdump use
-------------------------------------------------------------------------------
```c
> packet_setsockopt
>> packet_mc_add
>>> packet_dev_mc
>>>> dev_set_promiscuity
```

how ip command use
-------------------------------------------------------------------------------
***NOTE***
for netlink message:
the promisc flag is not store in dev->flags, but ***dev->gflags***.

so the 

```c
> rtnl_setlink
>> do_setlink
>>> dev_change_flags
>>>> __dev_change_flags
>>>>>4670         if ((flags ^ dev->gflags) & IFF_PROMISC) {
>>>>>4671                 int inc = (flags & IFF_PROMISC) ? 1 : -1;
>>>>>4672 
>>>>>4673                 dev->gflags ^= IFF_PROMISC;
>>>>>4674                 dev_set_promiscuity(dev, inc);
>>>>>4675         }
>>>> rtmsg_ifinfo       <=== netlink broadcast
>>>> __dev_notify_flags <=== callback notify
```

```c
2726 void __init rtnetlink_init(void)
2727 {
2728         if (register_pernet_subsys(&rtnetlink_net_ops))
2729                 panic("rtnetlink_init: cannot initialize rtnetlink\n");
2730
2731         register_netdevice_notifier(&rtnetlink_dev_notifier);
2732
2733         rtnl_register(PF_UNSPEC, RTM_GETLINK, rtnl_getlink,
2734                       rtnl_dump_ifinfo, rtnl_calcit);
2735         rtnl_register(PF_UNSPEC, RTM_SETLINK, rtnl_setlink, NULL, NULL);
2736         rtnl_register(PF_UNSPEC, RTM_NEWLINK, rtnl_newlink, NULL, NULL);
2737         rtnl_register(PF_UNSPEC, RTM_DELLINK, rtnl_dellink, NULL, NULL);
```

```c
1516 static int rtnl_setlink(struct sk_buff *skb, struct nlmsghdr *nlh)
...
1552         err = do_setlink(dev, ifm, tb, ifname, 0);
...
```

```c
1286 static int do_setlink(struct net_device *dev, struct ifinfomsg *ifm,
1287                       struct nlattr **tb, char *ifname, int modified)
...
1395         if (ifm->ifi_flags || ifm->ifi_change) {
1396                 err = dev_change_flags(dev, rtnl_dev_combine_flags(dev, ifm));
1397                 if (err < 0)
1398                         goto errout;
1399         }
```

```c
4715 int dev_change_flags(struct net_device *dev, unsigned int flags)
4716 {
4717         int ret;
4718         unsigned int changes, old_flags = dev->flags;
4719
4720         ret = __dev_change_flags(dev, flags);
4721         if (ret < 0)
4722                 return ret;
4723
4724         changes = old_flags ^ dev->flags;
4725         if (changes)
4726                 rtmsg_ifinfo(RTM_NEWLINK, dev, changes);
4727
4728         __dev_notify_flags(dev, old_flags);
4729         return ret;
4730 }
```
```c
4630 int __dev_change_flags(struct net_device *dev, unsigned int flags)
4631 {
4632         unsigned int old_flags = dev->flags;
4633         int ret;
4634
4635         ASSERT_RTNL();
4636
4637         /*
4638          *      Set the flags on our device.
4639          */
4640
4641         dev->flags = (flags & (IFF_DEBUG | IFF_NOTRAILERS | IFF_NOARP |
4642                                IFF_DYNAMIC | IFF_MULTICAST | IFF_PORTSEL |
4643                                IFF_AUTOMEDIA)) |
4644                      (dev->flags & (IFF_UP | IFF_VOLATILE | IFF_PROMISC |
4645                                     IFF_ALLMULTI));
4646
4647         /*
4648          *      Load in the correct multicast list now the flags have changed.
4649          */
4650
4651         if ((old_flags ^ flags) & IFF_MULTICAST)
4652                 dev_change_rx_flags(dev, IFF_MULTICAST);
4653
4654         dev_set_rx_mode(dev);
4655
4656         /*
4657          *      Have we downed the interface. We handle IFF_UP ourselves
4658          *      according to user attempts to set it, rather than blindly
4659          *      setting it.
4660          */
4661
4662         ret = 0;
4663         if ((old_flags ^ flags) & IFF_UP) {     /* Bit is different  ? */
4664                 ret = ((old_flags & IFF_UP) ? __dev_close : __dev_open)(dev);
4665
4666                 if (!ret)
4667                         dev_set_rx_mode(dev);
4668         }
4669
4670         if ((flags ^ dev->gflags) & IFF_PROMISC) {
4671                 int inc = (flags & IFF_PROMISC) ? 1 : -1;
4672
4673                 dev->gflags ^= IFF_PROMISC;
4674                 dev_set_promiscuity(dev, inc);
4675         }
4676
4677         /* NOTE: order of synchronization of IFF_PROMISC and IFF_ALLMULTI
4678            is important. Some (broken) drivers set IFF_PROMISC, when
4679            IFF_ALLMULTI is requested not asking us and not reporting.
4680          */
4681         if ((flags ^ dev->gflags) & IFF_ALLMULTI) {
4682                 int inc = (flags & IFF_ALLMULTI) ? 1 : -1;
4683
4684                 dev->gflags ^= IFF_ALLMULTI;
4685                 dev_set_allmulti(dev, inc);
4686         }
4687
4688         return ret;
4689 }
```
```
 648 static unsigned int rtnl_dev_combine_flags(const struct net_device *dev,
 649                                            const struct ifinfomsg *ifm)
 650 {
 651         unsigned int flags = ifm->ifi_flags;
 652
 653         /* bugwards compatibility: ifi_change == 0 is treated as ~0 */
 654         if (ifm->ifi_change)
 655                 flags = (flags & ifm->ifi_change) |
 656                         (rtnl_dev_get_flags(dev) & ~ifm->ifi_change);
 657
 658         return flags;
 659 }
```

---
layout: post
title: "how kernel add a IPv4 address"
date: 2013-08-20 16:31
comments: true
categories: [route]
tags: [IPv4, address, net]
---

##summary
There is a `struct in_device __rcu  *ip_ptr` under every net_device.
which stores all the IPv4 related information for this device.

Each IPv4 address is stored a `struct in_ifaddr`, 
all the IPv4 address of a net device are stored into a list(netdev->ip_ptr->ifa_list). 

<!-- more -->

```c
1075 struct net_device {
...
1208         struct in_device __rcu  *ip_ptr;        /* IPv4 specific data   */
```

```c
 55 struct in_device {
 56         struct net_device       *dev;
 57         atomic_t                refcnt;
 58         int                     dead;
 59         struct in_ifaddr        *ifa_list;      /* IP ifaddr chain              */
 ...
```

```c
161 struct in_ifaddr {
162         struct hlist_node       hash;
163         struct in_ifaddr        *ifa_next;
164         struct in_device        *ifa_dev;
165         struct rcu_head         rcu_head;
166         __be32                  ifa_local;
167         __be32                  ifa_address;
168         __be32                  ifa_mask;
169         __be32                  ifa_broadcast;
170         unsigned char           ifa_scope;
171         unsigned char           ifa_flags;
172         unsigned char           ifa_prefixlen;
173         char                    ifa_label[IFNAMSIZ];
174 
175         /* In seconds, relative to tstamp. Expiry is at tstamp + HZ * lft. */
176         __u32                   ifa_valid_lft;
177         __u32                   ifa_preferred_lft;
178         unsigned long           ifa_cstamp; /* created timestamp */
179         unsigned long           ifa_tstamp; /* updated timestamp */
180 };
```

```c
2284 void __init devinet_init(void)
2285 { 
...
2300         rtnl_register(PF_INET, RTM_NEWADDR, inet_rtm_newaddr, NULL, NULL);
2301         rtnl_register(PF_INET, RTM_DELADDR, inet_rtm_deladdr, NULL, NULL);
```


##calltrace 
```c
> inet_rtm_newaddr
> > rtm_to_ifaddr
> > find_matching_ifa
> > __inet_insert_ifa
> > > insert ifa to ifa_list with right postion.
> > > inet_hash_insert(dev_net(in_dev->dev), ifa);
> > > rtmsg_ifa(RTM_NEWADDR, ifa, nlh, portid);
> > > blocking_notifier_call_chain(&inetaddr_chain, NETDEV_UP, ifa);
```

####NOTE:
inetaddr_chain will be shown in next blog.


```c
 807 static int inet_rtm_newaddr(struct sk_buff *skb, struct nlmsghdr *nlh)
 808 {
 809         struct net *net = sock_net(skb->sk);
 810         struct in_ifaddr *ifa;
 811         struct in_ifaddr *ifa_existing;
 812         __u32 valid_lft = INFINITY_LIFE_TIME;
 813         __u32 prefered_lft = INFINITY_LIFE_TIME;
 814 
 815         ASSERT_RTNL();
 816 
 817         ifa = rtm_to_ifaddr(net, nlh, &valid_lft, &prefered_lft);
 818         if (IS_ERR(ifa))
 819                 return PTR_ERR(ifa);
 820 
 821         ifa_existing = find_matching_ifa(ifa);
 822         if (!ifa_existing) {
 823                 /* It would be best to check for !NLM_F_CREATE here but
 824                  * userspace alreay relies on not having to provide this.
 825                  */
 826                 set_ifa_lifetime(ifa, valid_lft, prefered_lft);
 827                 return __inet_insert_ifa(ifa, nlh, NETLINK_CB(skb).portid);
 828         } else {
 829                 inet_free_ifa(ifa);
 830 
 831                 if (nlh->nlmsg_flags & NLM_F_EXCL ||
 832                     !(nlh->nlmsg_flags & NLM_F_REPLACE))
 833                         return -EEXIST;
 834                 ifa = ifa_existing;
 835                 set_ifa_lifetime(ifa, valid_lft, prefered_lft);
 836                 cancel_delayed_work(&check_lifetime_work);
 837                 schedule_delayed_work(&check_lifetime_work, 0);
 838                 rtmsg_ifa(RTM_NEWADDR, ifa, nlh, NETLINK_CB(skb).portid);
 839                 blocking_notifier_call_chain(&inetaddr_chain, NETDEV_UP, ifa);
 840         }
 841         return 0;
 842 }
```

```c
 708 static struct in_ifaddr *rtm_to_ifaddr(struct net *net, struct nlmsghdr *nlh,
 709                                        __u32 *pvalid_lft, __u32 *pprefered_lft)
 710 {
 711         struct nlattr *tb[IFA_MAX+1];
 712         struct in_ifaddr *ifa;
 713         struct ifaddrmsg *ifm;
 714         struct net_device *dev;
 715         struct in_device *in_dev;
 716         int err;
 717 
 718         err = nlmsg_parse(nlh, sizeof(*ifm), tb, IFA_MAX, ifa_ipv4_policy);
 719         if (err < 0)
 720                 goto errout;
 721 
 722         ifm = nlmsg_data(nlh);
 723         err = -EINVAL;
 724         if (ifm->ifa_prefixlen > 32 || tb[IFA_LOCAL] == NULL)
 725                 goto errout;
 726 
 727         dev = __dev_get_by_index(net, ifm->ifa_index);
 728         err = -ENODEV;
 729         if (dev == NULL)
 730                 goto errout;
 731 
 732         in_dev = __in_dev_get_rtnl(dev);
 733         err = -ENOBUFS;
 734         if (in_dev == NULL)
 735                 goto errout;
 736 
 737         ifa = inet_alloc_ifa();
 738         if (ifa == NULL)
 739                 /*
 740                  * A potential indev allocation can be left alive, it stays
 741                  * assigned to its device and is destroy with it.
 742                  */
 743                 goto errout;
 744 
 745         ipv4_devconf_setall(in_dev);
 746         in_dev_hold(in_dev);
 747 
 748         if (tb[IFA_ADDRESS] == NULL)
 749                 tb[IFA_ADDRESS] = tb[IFA_LOCAL];
 750 
 751         INIT_HLIST_NODE(&ifa->hash);
 752         ifa->ifa_prefixlen = ifm->ifa_prefixlen;
 753         ifa->ifa_mask = inet_make_mask(ifm->ifa_prefixlen);
 754         ifa->ifa_flags = ifm->ifa_flags;
 755         ifa->ifa_scope = ifm->ifa_scope;
 756         ifa->ifa_dev = in_dev;
 757 
 758         ifa->ifa_local = nla_get_be32(tb[IFA_LOCAL]);
 759         ifa->ifa_address = nla_get_be32(tb[IFA_ADDRESS]);
 760 
 761         if (tb[IFA_BROADCAST])
 762                 ifa->ifa_broadcast = nla_get_be32(tb[IFA_BROADCAST]);
 763 
 764         if (tb[IFA_LABEL])
 765                 nla_strlcpy(ifa->ifa_label, tb[IFA_LABEL], IFNAMSIZ);
 766         else
 767                 memcpy(ifa->ifa_label, dev->name, IFNAMSIZ);
 768 
 769         if (tb[IFA_CACHEINFO]) {
 770                 struct ifa_cacheinfo *ci;
 771 
 772                 ci = nla_data(tb[IFA_CACHEINFO]);
 773                 if (!ci->ifa_valid || ci->ifa_prefered > ci->ifa_valid) {
 774                         err = -EINVAL;
 775                         goto errout_free;
 776                 }
 777                 *pvalid_lft = ci->ifa_valid;
 778                 *pprefered_lft = ci->ifa_prefered;
 779         }
 780 
 781         return ifa;
 782 
 783 errout_free:
 784         inet_free_ifa(ifa);
 785 errout:
 786         return ERR_PTR(err);
 787 }
```

```c
 426 static int __inet_insert_ifa(struct in_ifaddr *ifa, struct nlmsghdr *nlh,
 427                              u32 portid)
 428 {
 429         struct in_device *in_dev = ifa->ifa_dev;
 430         struct in_ifaddr *ifa1, **ifap, **last_primary;
 431 
 432         ASSERT_RTNL();
 433 
 434         if (!ifa->ifa_local) {
 435                 inet_free_ifa(ifa);
 436                 return 0;
 437         }
 438 
 439         ifa->ifa_flags &= ~IFA_F_SECONDARY;
 440         last_primary = &in_dev->ifa_list;
 441 
 442         for (ifap = &in_dev->ifa_list; (ifa1 = *ifap) != NULL;
 443              ifap = &ifa1->ifa_next) {
 444                 if (!(ifa1->ifa_flags & IFA_F_SECONDARY) &&
 445                     ifa->ifa_scope <= ifa1->ifa_scope)
 446                         last_primary = &ifa1->ifa_next;
 447                 if (ifa1->ifa_mask == ifa->ifa_mask &&
 448                     inet_ifa_match(ifa1->ifa_address, ifa)) {
 449                         if (ifa1->ifa_local == ifa->ifa_local) {
 450                                 inet_free_ifa(ifa);
 451                                 return -EEXIST;
 452                         }
 453                         if (ifa1->ifa_scope != ifa->ifa_scope) {
 454                                 inet_free_ifa(ifa);
 455                                 return -EINVAL;
 456                         }
 457                         ifa->ifa_flags |= IFA_F_SECONDARY;
 458                 }
 459         }
 460 
 461         if (!(ifa->ifa_flags & IFA_F_SECONDARY)) {
 462                 net_srandom(ifa->ifa_local);
 463                 ifap = last_primary;
 464         }
 465 
 466         ifa->ifa_next = *ifap;
 467         *ifap = ifa;
 468 
 469         inet_hash_insert(dev_net(in_dev->dev), ifa);
 470 
 471         cancel_delayed_work(&check_lifetime_work);
 472         schedule_delayed_work(&check_lifetime_work, 0);
 473 
 474         /* Send message first, then call notifier.
 475            Notifier will trigger FIB update, so that
 476            listeners of netlink will know about new ifaddr */
 477         rtmsg_ifa(RTM_NEWADDR, ifa, nlh, portid);
 478         blocking_notifier_call_chain(&inetaddr_chain, NETDEV_UP, ifa);
 479 
 480         return 0;
 481 }
```

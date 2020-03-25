---
layout: post
title: "Where is IPv6 route cache"
date: 2013-08-22 11:25
comments: true
categories: [route]
tags: [kernel, ipv6, route, cacahe]
---

## Q: Does kernel IPv6 route support route cache ?
 
  yes. but it is very different with IPv4.
  It should be 'clone' strictly speaking.

  For simple let us only focus on `ip6_pol_route`, and variable `struct rt6_info *nrt`.

1. When a route is matched, a `nrt` will be created by `rt6_alloc_cow` or `rt6_alloc_clone`.

2. the `nrt` will be set with flag `RTF_CACHE;`

3. `ip6_ins_rt` insert the new `nrt` to fib tree. 

4. kernel starts  fib6 gc for `nrt`.

so there will be a new `cache` route, and after a period it will disappear.

	
<!-- more -->

## Data structure

```c
 89 struct rt6_info {
 90         struct dst_entry                dst;     
 91    
 92         /*
 93          * Tail elements of dst_entry (__refcnt etc.)
 94          * and these elements (rarely used in hot path) are in
 95          * the same cache line.     
 96          */
 97         struct fib6_table               *rt6i_table;
 98         struct fib6_node                *rt6i_node;
 99    
100         struct in6_addr                 rt6i_gateway;
101    
102         /* Multipath routes:        
103          * siblings is a list of rt6_info that have the the same metric/weight,
104          * destination, but not the same gateway. nsiblings is just a cache
105          * to speed up lookup.      
106          */                         
107         struct list_head                rt6i_siblings;
108         unsigned int                    rt6i_nsiblings;
109    
110         atomic_t                        rt6i_ref;
111 
112         /* These are in a separate cache line. */
113         struct rt6key                   rt6i_dst ____cacheline_aligned_in_smp;   
114         u32                             rt6i_flags;
115         struct rt6key                   rt6i_src;
116         struct rt6key                   rt6i_prefsrc;
117         u32                             rt6i_metric;                             
118    
119         struct inet6_dev                *rt6i_idev;
120         unsigned long                   _rt6i_peer;
121    
122         u32                             rt6i_genid;
123 
124         /* more non-fragment space at head required */
125         unsigned short                  rt6i_nfheader_len;
126    
127         u8                              rt6i_protocol;                           
128 }; 
```
## call trace

Fox exmaple: forwarding a IPv6 packet.

```c
> ip6_rcv_finish
> > ip6_route_input
> > > ip6_route_input_lookup
> > > > fib6_rule_lookup(..., ip6_pol_route_input)
> > > > > fib_rules_lookup ( net->ipv6.fib6_rules_ops...)
> > > > > > ops->action <=== (fib6_rule_action)
> > > > > > > arg->lookup_ptr() <== ip6_pol_route_input
> > > > > > > > ip6_pol_route
> > > > > > > > > fn = fib6_lookup(&table->tb6_root, &fl6->daddr, &fl6->saddr);
> > > > > > > > > rt = rt6_select(fn, oif, strict | reachable);
> > > > > > > > > ip6_ins_rt(nrt) <=== !!!!
```
## methods

```c
 830 int ip6_ins_rt(struct rt6_info *rt)
 831 {
 832         struct nl_info info = {
 833                 .nl_net = dev_net(rt->dst.dev),
 834         };
 835         return __ip6_ins_rt(rt, &info);
 836 }
```

```c
 817 static int __ip6_ins_rt(struct rt6_info *rt, struct nl_info *info)
 818 {
 819         int err;
 820         struct fib6_table *table;
 821 
 822         table = rt->rt6i_table;
 823         write_lock_bh(&table->tb6_lock);
 824         err = fib6_add(&table->tb6_root, rt, info);
 825         write_unlock_bh(&table->tb6_lock);
 826 
 827         return err;
 828 }
```

```c
 809 int fib6_add(struct fib6_node *root, struct rt6_info *rt, struct nl_info *info)
 810 {
... 
 825         fn = fib6_add_1(root, &rt->rt6i_dst.addr, sizeof(struct in6_addr),
 826                         rt->rt6i_dst.plen, offsetof(struct rt6_info, rt6i_dst),
 827                         allow_create, replace_required);
 828 
 829         if (IS_ERR(fn)) {
 830                 err = PTR_ERR(fn);
 831                 goto out;
 832         }
 833 
 834         pn = fn;
...
 903         err = fib6_add_rt2node(fn, rt, info);
 904         if (!err) {
 905                 fib6_start_gc(info->nl_net, rt); <===
 906                 if (!(rt->rt6i_flags & RTF_CACHE))
 907                         fib6_prune_clones(info->nl_net, pn, rt);
 908         }
 909 
...
 946 }
 947 
```

## test log

### add IPv6 address to a interface.
```
➜  ~  sudo ip addr add dev eth0 3ffe::1/64
```

### check the related IPv6 route
```
➜  ~  sudo ip -6 route
3ffe::/64 dev eth0  proto kernel  metric 256 
fe80::/64 dev eth0  proto kernel  metric 256 
➜  ~  sudo ip -6 route show cache
➜  ~ 
```

### try to a ping6 test(expect failed).
```
➜  ~  ping6 3ffe::2
PING 3ffe::2(3ffe::2) 56 data bytes
^C
--- 3ffe::2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1007ms
```

### check the route again.
```
➜  ~  sudo ip -6 route show cache
3ffe::2 via 3ffe::2 dev eth0  metric 0 
    cache  expires -1sec
➜  ~
```

A `cache` route is inserted, while common route unchanged.

```
➜  ~  sudo ip -6 route show
3ffe::/64 dev eth0  proto kernel  metric 256 
fe80::/64 dev eth0  proto kernel  metric 256 
➜  ~
```
### after a period, check again. `cache` route disappear.
```
➜  ~  sudo ip -6 route show cache
➜  ~  sudo ip -6 route show
3ffe::/64 dev eth0  proto kernel  metric 256 
fe80::/64 dev eth0  proto kernel  metric 256 
➜  ~
```

## Serveral questions:
### Q1: What kernel will do with subnet route vs route cache?
   ex: 
	1. add route 3ffe::1/64
 	2. ping route 3ffe::2 a route cache will be insert to fib tree.
	3. add roue 3ffe::1/96
What kernel will do with it? the cache route will be deleted?
See following line in function `fib6_add`, kernel will prune all `clone/cache` route.
```c
 903         err = fib6_add_rt2node(fn, rt, info);
 904         if (!err) {
 905                 fib6_start_gc(info->nl_net, rt);
 906                 if (!(rt->rt6i_flags & RTF_CACHE))
 907                         fib6_prune_clones(info->nl_net, pn, rt); 
 908         }
```

### Q2: How to do with ecmp?



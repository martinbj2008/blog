---
layout: post
title: "How does IPV4/6 process input tcp/udp packet"
date: 2013-11-25 10:54
comments: true
categories: [socket] 
tags: [socket]
---

For the packet to localhost, `ip_local_deliver_finish` will be the 
last funciton called by network layer.

In `ip_local_deliver_finish`, it will be process the protocol
hander of a array element of `inet_protos` according the `protocol` value
in IPv4 header.

IPv6 is very similar vs IPv4 except the name is a bit different.

<!-- more -->
### Data structure for IPv4

#### `net_protocol`

```c
 34 /* This is one larger than the largest protocol value that can be
 35  * found in an ipv4 or ipv6 header.  Since in both cases the protocol
 36  * value is presented in a __u8, this is defined to be 256.
 37  */
 38 #define MAX_INET_PROTOS         256
 39 
 40 /* This is used to register protocols. */
 41 struct net_protocol {
 42         void                    (*early_demux)(struct sk_buff *skb);
 43         int                     (*handler)(struct sk_buff *skb);
 44         void                    (*err_handler)(struct sk_buff *skb, u32 info);
 45         unsigned int            no_policy:1,
 46                                 netns_ok:1;
 47 }
```

The `handler` is the main part, it will be called to process upper protol over IPv4.

Fox example: tcpv4 will be `tcp_v4_rcv`.
```
1540 static const struct net_protocol tcp_protocol = {
1541         .early_demux    =       tcp_v4_early_demux,
1542         .handler        =       tcp_v4_rcv,
1543         .err_handler    =       tcp_v4_err,
1544         .no_policy      =       1,
1545         .netns_ok       =       1,
1546 };
```

#### pointer array for all protocols

There is a pointer array `inet_protos` to collect all the supported protocols by IPv4.
```
 31 const struct net_protocol __rcu *inet_protos[MAX_INET_PROTOS] __read_mostly;
 32 const struct net_offload __rcu *inet_offloads[MAX_INET_PROTOS] __read_mostly;
```

The 'ADD' and 'DEL" operation is done by `inet_add_protocol` and `inet_del_protocol`

```
 34 /*
 35  *      Add a protocol handler to the hash tables
 36  */
 37 
 38 int inet_add_protocol(const struct net_protocol *prot, unsigned char protocol)
 39 {
 40         if (!prot->netns_ok) {
 41                 pr_err("Protocol %u is not namespace aware, cannot register.\n",
 42                         protocol);
 43                 return -EINVAL;
 44         }
 45 
 46         return !cmpxchg((const struct net_protocol **)&inet_protos[protocol],
 47                         NULL, prot) ? 0 : -1;
 48 }
 49 EXPORT_SYMBOL(inet_add_protocol);
```

```
 58 /*
 59  *      Remove a protocol from the hash tables.
 60  */
 61 
 62 int inet_del_protocol(const struct net_protocol *prot, unsigned char protocol)
 63 {
 64         int ret;
 65 
 66         ret = (cmpxchg((const struct net_protocol **)&inet_protos[protocol],
 67                        prot, NULL) == prot) ? 0 : -1;
 68 
 69         synchronize_net();
 70 
 71         return ret;
 72 }
 73 EXPORT_SYMBOL(inet_del_protocol);
 74 
```

All registered protocol handler.
```
$ grep inet_add_protocol net/ -Rw
net/dccp/ipv4.c:	err = inet_add_protocol(&dccp_v4_protocol, IPPROTO_DCCP);
net/sctp/protocol.c:	if (inet_add_protocol(&sctp_protocol, IPPROTO_SCTP) < 0)
net/l2tp/l2tp_ip.c:	err = inet_add_protocol(&l2tp_ip_protocol, IPPROTO_L2TP);
net/ipv4/ipmr.c:	if (inet_add_protocol(&pim_protocol, IPPROTO_PIM) < 0) {
net/ipv4/esp4.c:	if (inet_add_protocol(&esp4_protocol, IPPROTO_ESP) < 0) {
net/ipv4/protocol.c:int inet_add_protocol(const struct net_protocol *prot, unsigned char protocol)
net/ipv4/protocol.c:EXPORT_SYMBOL(inet_add_protocol);
net/ipv4/af_inet.c:	if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
net/ipv4/af_inet.c:	if (inet_add_protocol(&udp_protocol, IPPROTO_UDP) < 0)
net/ipv4/af_inet.c:	if (inet_add_protocol(&tcp_protocol, IPPROTO_TCP) < 0)
net/ipv4/af_inet.c:	if (inet_add_protocol(&igmp_protocol, IPPROTO_IGMP) < 0)
net/ipv4/ah4.c:	if (inet_add_protocol(&ah4_protocol, IPPROTO_AH) < 0) {
net/ipv4/tunnel4.c:	if (inet_add_protocol(&tunnel4_protocol, IPPROTO_IPIP)) {
net/ipv4/tunnel4.c:	if (inet_add_protocol(&tunnel64_protocol, IPPROTO_IPV6)) {
net/ipv4/ipcomp.c:	if (inet_add_protocol(&ipcomp4_protocol, IPPROTO_COMP) < 0) {
net/ipv4/udplite.c:	if (inet_add_protocol(&udplite_protocol, IPPROTO_UDPLITE) < 0)
net/ipv4/gre_demux.c:	if (inet_add_protocol(&net_gre_protocol, IPPROTO_GRE) < 0) {
```
#### regiter protocol handler for tcp, udp and icmp. 
```
1670 static int __init inet_init(void)
1671 {
...
1710         /*
1711          *      Add all the base protocols.
1712          */
1713 
1714         if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
1715                 pr_crit("%s: Cannot add ICMP protocol\n", __func__);
1716         if (inet_add_protocol(&udp_protocol, IPPROTO_UDP) < 0)
1717                 pr_crit("%s: Cannot add UDP protocol\n", __func__);
1718         if (inet_add_protocol(&tcp_protocol, IPPROTO_TCP) < 0)
1719                 pr_crit("%s: Cannot add TCP protocol\n", __func__);
...
```

#### call protocol handler in the end of IPv4 process.
NOTE:
1. `no_policy`: if it is set, `xfrm4_policy_check` will be
postponed and done by handler itself. such as `tcp`, `udp`.

```
190 static int ip_local_deliver_finish(struct sk_buff *skb)
191 {
192         struct net *net = dev_net(skb->dev);
193 
194         __skb_pull(skb, skb_network_header_len(skb));
195 
196         rcu_read_lock();
197         {
198                 int protocol = ip_hdr(skb)->protocol;
199                 const struct net_protocol *ipprot;
200                 int raw;
201 
202         resubmit:
203                 raw = raw_local_deliver(skb, protocol);
204 
205                 ipprot = rcu_dereference(inet_protos[protocol]);
206                 if (ipprot != NULL) {
207                         int ret;
208 
209                         if (!ipprot->no_policy) {
210                                 if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
211                                         kfree_skb(skb);
212                                         goto out;
213                                 }
214                                 nf_reset(skb);
215                         }
216                         ret = ipprot->handler(skb);
217                         if (ret < 0) {
218                                 protocol = -ret;
219                                 goto resubmit;
220                         }
221                         IP_INC_STATS_BH(net, IPSTATS_MIB_INDELIVERS);
222                 } else {
223                         if (!raw) {
224                                 if (xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
225                                         IP_INC_STATS_BH(net, IPSTATS_MIB_INUNKNOWNPROTOS);
226                                         icmp_send(skb, ICMP_DEST_UNREACH,
227                                                   ICMP_PROT_UNREACH, 0);
228                                 }
229                                 kfree_skb(skb);
230                         } else {
231                                 IP_INC_STATS_BH(net, IPSTATS_MIB_INDELIVERS);
232                                 consume_skb(skb);
233                         }
234                 }
235         }
236  out:
237         rcu_read_unlock();
238 
239         return 0;
240 }
```


### IPV6 part
#### `struct inet6_protocol`
```
 49 #if IS_ENABLED(CONFIG_IPV6)
 50 struct inet6_protocol {
 51         void    (*early_demux)(struct sk_buff *skb);
 52 
 53         int     (*handler)(struct sk_buff *skb);
 54 
 55         void    (*err_handler)(struct sk_buff *skb,
 56                                struct inet6_skb_parm *opt,
 57                                u8 type, u8 code, int offset,
 58                                __be32 info);
 59         unsigned int    flags;  /* INET6_PROTO_xxx */
 60 };
 61 
 62 #define INET6_PROTO_NOPOLICY    0x1
 63 #define INET6_PROTO_FINAL       0x2
 64 #endif
```
#### ADD and DEL
```
 29 const struct inet6_protocol __rcu *inet6_protos[MAX_INET_PROTOS] __read_mostly;
 30 EXPORT_SYMBOL(inet6_protos);
 31 
 32 int inet6_add_protocol(const struct inet6_protocol *prot, unsigned char protocol)
 33 {
 34         return !cmpxchg((const struct inet6_protocol **)&inet6_protos[protocol],
 35                         NULL, prot) ? 0 : -1;
 36 }
 37 EXPORT_SYMBOL(inet6_add_protocol);
 38 
 39 /*
 40  *      Remove a protocol from the hash tables.
 41  */
 42 
 43 int inet6_del_protocol(const struct inet6_protocol *prot, unsigned char protocol)
 44 {
 45         int ret;
 46 
 47         ret = (cmpxchg((const struct inet6_protocol **)&inet6_protos[protocol],
 48                        prot, NULL) == prot) ? 0 : -1;
 49 
 50         synchronize_net();
 51 
 52         return ret;
 53 }
 54 EXPORT_SYMBOL(inet6_del_protocol);
```

#### protocol handlers
```
$ grep inet6_add_protocol net/ -Rw
net/dccp/ipv6.c:	err = inet6_add_protocol(&dccp_v6_protocol, IPPROTO_DCCP);
net/sctp/ipv6.c:	if (inet6_add_protocol(&sctpv6_protocol, IPPROTO_SCTP) < 0)
net/ipv6/exthdrs.c:	ret = inet6_add_protocol(&rthdr_protocol, IPPROTO_ROUTING);
net/ipv6/exthdrs.c:	ret = inet6_add_protocol(&destopt_protocol, IPPROTO_DSTOPTS);
net/ipv6/exthdrs.c:	ret = inet6_add_protocol(&nodata_protocol, IPPROTO_NONE);
net/ipv6/icmp.c:	if (inet6_add_protocol(&icmpv6_protocol, IPPROTO_ICMPV6) < 0)
net/ipv6/ip6mr.c:	if (inet6_add_protocol(&pim6_protocol, IPPROTO_PIM) < 0) {
net/ipv6/esp6.c:	if (inet6_add_protocol(&esp6_protocol, IPPROTO_ESP) < 0) {
net/ipv6/protocol.c:int inet6_add_protocol(const struct inet6_protocol *prot, unsigned char protocol)
net/ipv6/protocol.c:EXPORT_SYMBOL(inet6_add_protocol);
net/ipv6/ip6_gre.c:	err = inet6_add_protocol(&ip6gre_protocol, IPPROTO_GRE);
net/ipv6/reassembly.c:	ret = inet6_add_protocol(&frag_protocol, IPPROTO_FRAGMENT);
net/ipv6/ipcomp6.c:	if (inet6_add_protocol(&ipcomp6_protocol, IPPROTO_COMP) < 0) {
net/ipv6/udp.c:	ret = inet6_add_protocol(&udpv6_protocol, IPPROTO_UDP);
net/ipv6/ah6.c:	if (inet6_add_protocol(&ah6_protocol, IPPROTO_AH) < 0) {
net/ipv6/tcp_ipv6.c:	ret = inet6_add_protocol(&tcpv6_protocol, IPPROTO_TCP);
net/ipv6/udplite.c:	ret = inet6_add_protocol(&udplitev6_protocol, IPPROTO_UDPLITE);
net/ipv6/tunnel6.c:	if (inet6_add_protocol(&tunnel6_protocol, IPPROTO_IPV6)) {
net/ipv6/tunnel6.c:	if (inet6_add_protocol(&tunnel46_protocol, IPPROTO_IPIP)) {
net/l2tp/l2tp_ip6.c:	err = inet6_add_protocol(&l2tp_ip6_protocol, IPPROTO_L2TP);
```

```
1951 static const struct inet6_protocol tcpv6_protocol = {
1952         .early_demux    =       tcp_v6_early_demux,
1953         .handler        =       tcp_v6_rcv,
1954         .err_handler    =       tcp_v6_err,
1955         .flags          =       INET6_PROTO_NOPOLICY|INET6_PROTO_FINAL,
1956 };
```
#### call protocol handler in the end of ipv6 process.
```
200 
201 static int ip6_input_finish(struct sk_buff *skb)
202 {
...
222         raw = raw6_local_deliver(skb, nexthdr);
223         if ((ipprot = rcu_dereference(inet6_protos[nexthdr])) != NULL) {
224                 int ret;
225 
226                 if (ipprot->flags & INET6_PROTO_FINAL) {
227                         const struct ipv6hdr *hdr;
228 
229                         /* Free reference early: we don't need it any more,
230                            and it may hold ip_conntrack module loaded
231                            indefinitely. */
232                         nf_reset(skb);
233 
234                         skb_postpull_rcsum(skb, skb_network_header(skb),
235                                            skb_network_header_len(skb));
236                         hdr = ipv6_hdr(skb);
237                         if (ipv6_addr_is_multicast(&hdr->daddr) &&
238                             !ipv6_chk_mcast_addr(skb->dev, &hdr->daddr,
239                             &hdr->saddr) &&
240                             !ipv6_is_mld(skb, nexthdr, skb_network_header_len(skb)))
241                                 goto discard;
242                 }
243                 if (!(ipprot->flags & INET6_PROTO_NOPOLICY) &&
244                     !xfrm6_policy_check(NULL, XFRM_POLICY_IN, skb))
245                         goto discard;
246 
247                 ret = ipprot->handler(skb);
248                 if (ret > 0)
249                         goto resubmit;
250                 else if (ret == 0)
251                         IP6_INC_STATS_BH(net, idev, IPSTATS_MIB_INDELIVERS);
252         } else {
...
```

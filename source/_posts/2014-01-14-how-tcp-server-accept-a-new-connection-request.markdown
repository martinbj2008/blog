---
layout: post
title: "how does tcp server accept a new connection request"
date: 2014-01-14 16:01
comments: true
categories: [socket]
tags: [socket]
---

Two packets will be proessed by tcp server side:
1. SYN
2. ACK

For the first packet(syn) of handshake, it first lookup the listen socket,
and create a req socket as temporary.

For the third packet(ack) of handshake, it will match the req socket created
in previous steps.
<!-- more -->

SYN:  
```c
== tcp_v4_rcv(struct sk_buff *skb)
== ==  sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest); 
== ==  tcp_v4_do_rcv(sk, skb);
== == == struct sock *nsk = tcp_v4_hnd_req(sk, skb)
== == == tcp_rcv_state_process
== == == == if (icsk->icsk_af_ops->conn_request(sk, skb) < 0)
== == == == tcp_v4_conn_request: icsk->icsk_af_ops->conn_request
== == == == == inet_reqsk_alloc
== == == == == tcp_openreq_init(req, &tmp_opt, skb);
== == == == == inet_csk_reqsk_queue_hash_add 
== == == == == == reqsk_queue_hash_req
== == == == == == inet_csk_reqsk_queue_added
```

ACK:
###`tcp_rcv_state_process` use a child socket, not the listen socket.
The new socket state is `TCP_SYN_RECV`

```c
== tcp_v4_rcv(struct sk_buff *skb)
== ==  sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest); 
== == tcp_v4_do_rcv(sk, skb);
== == == nsk = tcp_v4_hnd_req(sk, skb)
== == == == inet_csk_search_req
== == == == tcp_check_req
== == == == == child = inet_csk(sk)->icsk_af_ops->syn_recv_sock(sk, skb, req, NULL);
== == == == == inet_csk_reqsk_queue_unlink(sk, req, prev);
== == == == == inet_csk_reqsk_queue_removed(sk, req);
== == == == == inet_csk_reqsk_queue_add(sk, req, child);
== == == tcp_child_process(sk, nsk, skb)
== == == ==  tcp_rcv_state_process(child, skb, tcp_hdr(skb),...i
== == == == TCP_SYN_RECV: tcp_rcv_synsent_state_process
```

### `tcp_v4_rcv`
```c
1936 int tcp_v4_rcv(struct sk_buff *skb)
1937 {
1938         const struct iphdr *iph;
1939         const struct tcphdr *th;
1940         struct sock *sk;
1941         int ret;
1942         struct net *net = dev_net(skb->dev);
1943
1944         if (skb->pkt_type != PACKET_HOST)
1945                 goto discard_it;
1946
1947         /* Count it even if it's bad */
1948         TCP_INC_STATS_BH(net, TCP_MIB_INSEGS);
1949
1950         if (!pskb_may_pull(skb, sizeof(struct tcphdr)))
1951                 goto discard_it;
1952
1953         th = tcp_hdr(skb);
1954
1955         if (th->doff < sizeof(struct tcphdr) / 4)
1956                 goto bad_packet;
1957         if (!pskb_may_pull(skb, th->doff * 4))
1958                 goto discard_it;
1959
1960         /* An explanation is required here, I think.
1961          * Packet length and doff are validated by header prediction,
1962          * provided case of th->doff==0 is eliminated.
1963          * So, we defer the checks. */
1964         if (!skb_csum_unnecessary(skb) && tcp_v4_checksum_init(skb))
1965                 goto csum_error;
1966
1967         th = tcp_hdr(skb);
1968         iph = ip_hdr(skb);
1969         TCP_SKB_CB(skb)->seq = ntohl(th->seq);
1970         TCP_SKB_CB(skb)->end_seq = (TCP_SKB_CB(skb)->seq + th->syn + th->fin +
1971                                     skb->len - th->doff * 4);
1972         TCP_SKB_CB(skb)->ack_seq = ntohl(th->ack_seq);
1973         TCP_SKB_CB(skb)->when    = 0;
1974         TCP_SKB_CB(skb)->ip_dsfield = ipv4_get_dsfield(iph);
1975         TCP_SKB_CB(skb)->sacked  = 0;
1976
1977         sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest);
1978         if (!sk)
1979                 goto no_tcp_socket;
1980
1981 process:
1982         if (sk->sk_state == TCP_TIME_WAIT)
1983                 goto do_time_wait;
1984
1985         if (unlikely(iph->ttl < inet_sk(sk)->min_ttl)) {
1986                 NET_INC_STATS_BH(net, LINUX_MIB_TCPMINTTLDROP);
1987                 goto discard_and_relse;
1988         }
1989
1990         if (!xfrm4_policy_check(sk, XFRM_POLICY_IN, skb))
1991                 goto discard_and_relse;
1992         nf_reset(skb);
1993
1994         if (sk_filter(sk, skb))
1995                 goto discard_and_relse;
1996
1997         sk_mark_napi_id(sk, skb);
1998         skb->dev = NULL;
1999
2000         bh_lock_sock_nested(sk);
2001         ret = 0;
2002         if (!sock_owned_by_user(sk)) {
2003 #ifdef CONFIG_NET_DMA
2004                 struct tcp_sock *tp = tcp_sk(sk);
2005                 if (!tp->ucopy.dma_chan && tp->ucopy.pinned_list)
2006                         tp->ucopy.dma_chan = net_dma_find_channel();
2007                 if (tp->ucopy.dma_chan)
2008                         ret = tcp_v4_do_rcv(sk, skb);
2009                 else
2010 #endif
2011                 {
2012                         if (!tcp_prequeue(sk, skb))
2013                                 ret = tcp_v4_do_rcv(sk, skb);
2014                 }
2015         } else if (unlikely(sk_add_backlog(sk, skb,
2016                                            sk->sk_rcvbuf + sk->sk_sndbuf))) {
2017                 bh_unlock_sock(sk);
2018                 NET_INC_STATS_BH(net, LINUX_MIB_TCPBACKLOGDROP);
2019                 goto discard_and_relse;
2020         }
2021         bh_unlock_sock(sk);
2022
2023         sock_put(sk);
2024
2025         return ret;
2026
2027 no_tcp_socket:
2028         if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb))
2029                 goto discard_it;
2030
2031         if (skb->len < (th->doff << 2) || tcp_checksum_complete(skb)) {
2032 csum_error:
2033                 TCP_INC_STATS_BH(net, TCP_MIB_CSUMERRORS);
2034 bad_packet:
2035                 TCP_INC_STATS_BH(net, TCP_MIB_INERRS);
2036         } else {
2037                 tcp_v4_send_reset(NULL, skb);
2038         }
2039
2040 discard_it:
2041         /* Discard frame. */
2042         kfree_skb(skb);
2043         return 0;
2044
2045 discard_and_relse:
2046         sock_put(sk);
2047         goto discard_it;
2048
2049 do_time_wait:
2050         if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
2051                 inet_twsk_put(inet_twsk(sk));
2052                 goto discard_it;
2053         }
2054
2055         if (skb->len < (th->doff << 2)) {
2056                 inet_twsk_put(inet_twsk(sk));
2057                 goto bad_packet;
2058         }
2059         if (tcp_checksum_complete(skb)) {
2060                 inet_twsk_put(inet_twsk(sk));
2061                 goto csum_error;
2062         }
2063         switch (tcp_timewait_state_process(inet_twsk(sk), skb, th)) {
2064         case TCP_TW_SYN: {
2065                 struct sock *sk2 = inet_lookup_listener(dev_net(skb->dev),
2066                                                         &tcp_hashinfo,
2067                                                         iph->saddr, th->source,
2068                                                         iph->daddr, th->dest,
2069                                                         inet_iif(skb));
2070                 if (sk2) {
2071                         inet_twsk_deschedule(inet_twsk(sk), &tcp_death_row);
2072                         inet_twsk_put(inet_twsk(sk));
2073                         sk = sk2;
2074                         goto process;
2075                 }
2076                 /* Fall through to ACK */
2077         }
2078         case TCP_TW_ACK:
2079                 tcp_v4_timewait_ack(sk, skb);
2080                 break;
2081         case TCP_TW_RST:
2082                 goto no_tcp_socket;
2083         case TCP_TW_SUCCESS:;
2084         }
2085         goto discard_it;
2086 }
```
###`tcp_v4_do_rcv`
```c
1778 int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
1779 {
1780         struct sock *rsk;
1781 #ifdef CONFIG_TCP_MD5SIG
1782         /*
1783          * We really want to reject the packet as early as possible
1784          * if:
1785          *  o We're expecting an MD5'd packet and this is no MD5 tcp option
1786          *  o There is an MD5 option and we're not expecting one
1787          */
1788         if (tcp_v4_inbound_md5_hash(sk, skb))
1789                 goto discard;
1790 #endif
1791
1792         if (sk->sk_state == TCP_ESTABLISHED) { /* Fast path */
1793                 struct dst_entry *dst = sk->sk_rx_dst;
1794
1795                 sock_rps_save_rxhash(sk, skb);
1796                 if (dst) {
1797                         if (inet_sk(sk)->rx_dst_ifindex != skb->skb_iif ||
1798                             dst->ops->check(dst, 0) == NULL) {
1799                                 dst_release(dst);
1800                                 sk->sk_rx_dst = NULL;
1801                         }
1802                 }
1803                 tcp_rcv_established(sk, skb, tcp_hdr(skb), skb->len);
1804                 return 0;
1805         }
1806
1807         if (skb->len < tcp_hdrlen(skb) || tcp_checksum_complete(skb))
1808                 goto csum_err;
1809
1810         if (sk->sk_state == TCP_LISTEN) {
1809
1810         if (sk->sk_state == TCP_LISTEN) {
1811                 struct sock *nsk = tcp_v4_hnd_req(sk, skb);
1812                 if (!nsk)
1813                         goto discard;
1814
1815                 if (nsk != sk) {
1816                         sock_rps_save_rxhash(nsk, skb);
1817                         if (tcp_child_process(sk, nsk, skb)) {
1818                                 rsk = nsk;
1819                                 goto reset;
1820                         }
1821                         return 0;
1822                 }
1823         } else
1824                 sock_rps_save_rxhash(sk, skb);
1825
1826         if (tcp_rcv_state_process(sk, skb, tcp_hdr(skb), skb->len)) {
1827                 rsk = sk;
1828                 goto reset;
1829         }
1830         return 0;
1831
1832 reset:
1833         tcp_v4_send_reset(rsk, skb);
1834 discard:
1835         kfree_skb(skb);
1836         /* Be careful here. If this function gets more complicated and
1837          * gcc suffers from register pressure on the x86, sk (in %ebx)
1838          * might be destroyed here. This current version compiles correctly,
1839          * but you have been warned.
1840          */
1841         return 0;
1842
1843 csum_err:
1844         TCP_INC_STATS_BH(sock_net(sk), TCP_MIB_CSUMERRORS);
1845         TCP_INC_STATS_BH(sock_net(sk), TCP_MIB_INERRS);
1846         goto discard;
1847 }
1848 EXPORT_SYMBOL(tcp_v4_do_rcv);
```

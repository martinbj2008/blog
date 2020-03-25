---
layout: post
title: "Tcp Checksum in Send Direction"
date: 2012-07-13 00:00
comments: true
categories: [socket]
tags: [kernel, checksum, tcp, socket]
---

##summary
  When nic supports hardware checksum, tcp only partially calculate the sum and fill the related info into skb three item:

  1. `ip_summed`
  2. `csum_start`
  3. `csum_offset`

when nic driver sends the packet, it will fill these information to hardware’s correspond register.

<!-- more -->
##call trace

```c
> tcp_sendmsg
> > __tcp_push_pending_frames tcp_push_one
> > > tcp_write_xmit tcp_transmit_skb
> > > > icsk->icsk_af_ops->send_check(sk, skb);
> > > > > err = icsk->icsk_af_ops->queue_xmit(skb, &inet->cork.fl);
```

注：`__tcp_push_pending_frames`和 `tcp_push_one`最终都调用`tcp_write_xmit`

##relate source

###`tcp_prot` and `tcp_sendmsg`

```
 2612 struct proto tcp_prot = {
...
 2620         .init                   = tcp_v4_init_sock,
 2626         .sendmsg                = tcp_sendmsg,
```
###tcp_sendmsg
```c
 916 int tcp_sendmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,
 917                 size_t size)
 918 {
...
 987    if (sk->sk_route_caps & NETIF_F_ALL_CSUM)
 988       skb->ip_summed = CHECKSUM_PARTIAL;
 989
....
1096                         if (forced_push(tp)) {
1097                                 tcp_mark_push(tp, skb);
1098                                 __tcp_push_pending_frames(sk, mss_now, TCP_NAGLE_PUSH);
1099                         } else if (skb == tcp_send_head(sk))
1100                                 tcp_push_one(sk, mss_now);
__tcp_push_pending_frames
```

```c
1828 void __tcp_push_pending_frames(struct sock *sk, unsigned int cur_mss,
1829                                int nonagle)
1830 {
1831         /* If we are closed, the bytes will have to remain here.
1832          * In time closedown will finish, we empty the write queue and
1833          * all will be happy.
1834          */
1835         if (unlikely(sk->sk_state == TCP_CLOSE))
1836                 return;
1837
1838         if (tcp_write_xmit(sk, cur_mss, nonagle, 0, GFP_ATOMIC))
1839                 tcp_check_probe_timer(sk);
1840 }
```
###tcp_push_one
```
1845 void tcp_push_one(struct sock *sk, unsigned int mss_now)
1846 {
1847         struct sk_buff *skb = tcp_send_head(sk);
1848
1849         BUG_ON(!skb || skb->len < mss_now);
1850
1851         tcp_write_xmit(sk, mss_now, TCP_NAGLE_PUSH, 1, sk->sk_allocation);
1852 }
```
###tcp_write_xmit
```c
1743 static int tcp_write_xmit(struct sock *sk, unsigned int mss_now, int nonagle,
1744                           int push_one, gfp_t gfp)
1745 {
...
1800                 if (unlikely(tcp_transmit_skb(sk, skb, 1, gfp)))
1801                         break;
```
###tcp_transmit_skb
```c
 796 static int tcp_transmit_skb(struct sock *sk, struct sk_buff *skb, int clone_it,
 797                             gfp_t gfp_mask)
 798 {
 799         const struct inet_connection_sock *icsk = inet_csk(sk);
...
 892         icsk->icsk_af_ops->send_check(sk, skb);
...
 904         err = icsk->icsk_af_ops->queue_xmit(skb, &inet->cork.fl);
```
###tcp_v4_init_sock
```c
1875 static int tcp_v4_init_sock(struct sock *sk)
1876 {
1877         struct inet_connection_sock *icsk = inet_csk(sk);
1878         struct tcp_sock *tp = tcp_sk(sk);
1879
1880         skb_queue_head_init(&tp->out_of_order_queue);
1881         tcp_init_xmit_timers(sk);
1882         tcp_prequeue_init(tp);
1883
1884         icsk->icsk_rto = TCP_TIMEOUT_INIT;
1885         tp->mdev = TCP_TIMEOUT_INIT;
1886
1887         /* So many TCP implementations out there (incorrectly) count the
1888          * initial SYN frame in their delayed-ACK and congestion control
1889          * algorithms that we must have the following bandaid to talk
1890          * efficiently to them.  -DaveM
1891          */
1892         tp->snd_cwnd = TCP_INIT_CWND;
1893
1894         /* See draft-stevens-tcpca-spec-01 for discussion of the
1895          * initialization of these values.
1896          */
1897         tp->snd_ssthresh = TCP_INFINITE_SSTHRESH;
1898         tp->snd_cwnd_clamp = ~0;
1899         tp->mss_cache = TCP_MSS_DEFAULT;
1900
1901         tp->reordering = sysctl_tcp_reordering;
1902         icsk->icsk_ca_ops = &tcp_init_congestion_ops;
1903
1904         sk->sk_state = TCP_CLOSE;
1905
1906         sk->sk_write_space = sk_stream_write_space;
1907         sock_set_flag(sk, SOCK_USE_WRITE_QUEUE);
1908
1909         icsk->icsk_af_ops = &ipv4_specific;
1910         icsk->icsk_sync_mss = tcp_sync_mss;
```
###inet_connection_sock_af_ops
```c
1844 const struct inet_connection_sock_af_ops ipv4_specific = {
1845         .queue_xmit        = ip_queue_xmit,
1846         .send_check        = tcp_v4_send_check,
1847         .rebuild_header    = inet_sk_rebuild_header,
1848         .conn_request      = tcp_v4_conn_request,
1849         .syn_recv_sock     = tcp_v4_syn_recv_sock,
1850         .get_peer          = tcp_v4_get_peer,
1851         .net_header_len    = sizeof(struct iphdr),
1852         .setsockopt        = ip_setsockopt,
1853         .getsockopt        = ip_getsockopt,
1854         .addr2sockaddr     = inet_csk_addr2sockaddr,
1855         .sockaddr_len      = sizeof(struct sockaddr_in),
1856         .bind_conflict     = inet_csk_bind_conflict,
1857 #ifdef CONFIG_COMPAT
1858         .compat_setsockopt = compat_ip_setsockopt,
1859         .compat_getsockopt = compat_ip_getsockopt,
1860 #endif
1861 };
```

###tcp_v4_send_check
```c
 546 /* This routine computes an IPv4 TCP checksum. */
 547 void tcp_v4_send_check(struct sock *sk, struct sk_buff *skb)
 548 {
 549         const struct inet_sock *inet = inet_sk(sk);
 550
 551         __tcp_v4_send_check(skb, inet->inet_saddr, inet->inet_daddr);
 552 }
```
###`__tcp_v4_send_check`
```c
 529 static void __tcp_v4_send_check(struct sk_buff *skb,
 530                                 __be32 saddr, __be32 daddr)
 531 {
 532         struct tcphdr *th = tcp_hdr(skb);
 533
 534         if (skb->ip_summed == CHECKSUM_PARTIAL) {
 535                 th->check = ~tcp_v4_check(skb->len, saddr, daddr, 0);
 536                 skb->csum_start = skb_transport_header(skb) - skb->head;
 537                 skb->csum_offset = offsetof(struct tcphdr, check);
 538         } else {
 539                 th->check = tcp_v4_check(skb->len, saddr, daddr,
 540                                          csum_partial(th,
 541                                                       th->doff << 2,
 542                                                       skb->csum));
 543         }
 544 }
```
neigh operations
```c
 129 static const struct neigh_ops arp_generic_ops = {
 130         .family =               AF_INET,
 131         .solicit =              arp_solicit,
 132         .error_report =         arp_error_report,
 133         .output =               neigh_resolve_output,
 134         .connected_output =     neigh_connected_output,
 135 };
 136
 137 static const struct neigh_ops arp_hh_ops = {
 138         .family =               AF_INET,
 139         .solicit =              arp_solicit,
 140         .error_report =         arp_error_report,
 141         .output =               neigh_resolve_output,
 142         .connected_output =     neigh_resolve_output,
 143 };
 144
 145 static const struct neigh_ops arp_direct_ops = {
 146         .family =               AF_INET,
 147         .output =               neigh_direct_output,
 148         .connected_output =     neigh_direct_output,
 149 };
 150
 151 static const struct neigh_ops arp_broken_ops = {
 152         .family =               AF_INET,
 153         .solicit =              arp_solicit,
 154         .error_report =         arp_error_report,
 155         .output =               neigh_compat_output,
 156         .connected_output =     neigh_compat_output,
 157 };
```

```c
1280 int neigh_resolve_output(struct neighbour *neigh, struct sk_buff *skb)
1281 {
...
1304                 if (err >= 0)
1305                         rc = dev_queue_xmit(skb);
```
```c
2511 int dev_queue_xmit(struct sk_buff *skb)
2512 {
...
2561                                 rc = dev_hard_start_xmit(skb, dev, txq);
```

###`dev_hard_start_xmit`
```c
2184 int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev,
2185                         struct netdev_queue *txq)
2186 {
...
2204                 features = netif_skb_features(skb);
...
2215                 if (netif_needs_gso(skb, features)) {
2216                         if (unlikely(dev_gso_segment(skb, features)))
2217                                 goto out_kfree_skb;
2218                         if (skb->next)
2219                                 goto gso;
2220                 } else {
2221                         if (skb_needs_linearize(skb, features) &&
2222                             __skb_linearize(skb))
2223                                 goto out_kfree_skb;
2224
2225                         /* If packet is not checksummed and device does not
2226                          * support checksumming for this protocol, complete
2227                          * checksumming here.
2228                          */
2229                         if (skb->ip_summed == CHECKSUM_PARTIAL) {
2230                                 skb_set_transport_header(skb,
2231                                         skb_checksum_start_offset(skb));
2232                                 if (!(features & NETIF_F_ALL_CSUM) &&
2233                                      skb_checksum_help(skb))
2234                                         goto out_kfree_skb;
2235                         }
2236                 }
...
2239                 rc = ops->ndo_start_xmit(skb, dev);
```
###nic dirver e1000
```c
 847 static const struct net_device_ops e1000_netdev_ops = {
 848         .ndo_open               = e1000_open,
 849         .ndo_stop               = e1000_close,
 850         .ndo_start_xmit         = e1000_xmit_frame,
 851         .ndo_get_stats          = e1000_get_stats,
 852         .ndo_set_rx_mode        = e1000_set_rx_mode,
 853         .ndo_set_mac_address    = e1000_set_mac,
 854         .ndo_tx_timeout         = e1000_tx_timeout,
 855         .ndo_change_mtu         = e1000_change_mtu,
 856         .ndo_do_ioctl           = e1000_ioctl,
 857         .ndo_validate_addr      = eth_validate_addr,
 858         .ndo_vlan_rx_add_vid    = e1000_vlan_rx_add_vid,
 859         .ndo_vlan_rx_kill_vid   = e1000_vlan_rx_kill_vid,
 860 #ifdef CONFIG_NET_POLL_CONTROLLER
 861         .ndo_poll_controller    = e1000_netpoll,
 862 #endif
 863         .ndo_fix_features       = e1000_fix_features,
 864         .ndo_set_features       = e1000_set_features,
 865 };
```

```c
5013 static netdev_tx_t e1000_xmit_frame(struct sk_buff *skb,
5014                                     struct net_device *netdev)
5015 {
...
5113         else if (e1000_tx_csum(tx_ring, skb))
5114                 tx_flags |= E1000_TX_FLAGS_CSUM;
```
```c
2781 static bool e1000_tx_csum(struct e1000_adapter *adapter,
2782                           struct e1000_tx_ring *tx_ring, struct sk_buff *skb)
2783 {
2784         struct e1000_context_desc *context_desc;
2785         struct e1000_buffer *buffer_info;
2786         unsigned int i;
2787         u8 css;
2788         u32 cmd_len = E1000_TXD_CMD_DEXT;
2789
2790         if (skb->ip_summed != CHECKSUM_PARTIAL)
2791                 return false;
2792
2793         switch (skb->protocol) {
2794         case cpu_to_be16(ETH_P_IP):
2795                 if (ip_hdr(skb)->protocol == IPPROTO_TCP)
2796                         cmd_len |= E1000_TXD_CMD_TCP;
2797                 break;
2798         case cpu_to_be16(ETH_P_IPV6):
2799                 /* XXX not handling all IPV6 headers */
2800                 if (ipv6_hdr(skb)->nexthdr == IPPROTO_TCP)
2801                         cmd_len |= E1000_TXD_CMD_TCP;
2802                 break;
2803         default:
2804                 if (unlikely(net_ratelimit()))
2805                         e_warn(drv, "checksum_partial proto=%x!\n",
2806                                skb->protocol);
2807                 break;
2808         }
2809
2810         css = skb_checksum_start_offset(skb);
2811
2812         i = tx_ring->next_to_use;
2813         buffer_info = &tx_ring->buffer_info[i];
2814         context_desc = E1000_CONTEXT_DESC(*tx_ring, i);
2815
2816         context_desc->lower_setup.ip_config = 0;
2817         context_desc->upper_setup.tcp_fields.tucss = css;
2818         context_desc->upper_setup.tcp_fields.tucso =
2819                 css + skb->csum_offset;
2820         context_desc->upper_setup.tcp_fields.tucse = 0;
2821         context_desc->tcp_seg_setup.data = 0;
2822         context_desc->cmd_and_length = cpu_to_le32(cmd_len);
2823
2824         buffer_info->time_stamp = jiffies;
2825         buffer_info->next_to_watch = i;
2826
2827         if (unlikely(++i == tx_ring->count)) i = 0;
```
```c
 201 static struct pci_driver e1000_driver = {
 202         .name     = e1000_driver_name,
 203         .id_table = e1000_pci_tbl,
 204         .probe    = e1000_probe,
 205         .remove   = __devexit_p(e1000_remove),
 206 #ifdef CONFIG_PM
 207         /* Power Management Hooks */
 208         .suspend  = e1000_suspend,
 209         .resume   = e1000_resume,
 210 #endif
 211         .shutdown = e1000_shutdown,
 212         .err_handler = &e1000_err_handler
 213 };
```
```c
 942 static int __devinit e1000_probe(struct pci_dev *pdev,
 943                                  const struct pci_device_id *ent)
 944 {
 945         struct net_device *netdev;
 946         struct e1000_adapter *adapter;
...
1065         if (hw->mac_type >= e1000_82543) {
1066                 netdev->hw_features = NETIF_F_SG |
1067                                    NETIF_F_HW_CSUM |
1068                                    NETIF_F_HW_VLAN_RX;
1069                 netdev->features = NETIF_F_HW_VLAN_TX |
1070                                    NETIF_F_HW_VLAN_FILTER;
1071         }
1072
1073         if ((hw->mac_type >= e1000_82544) &&
1074            (hw->mac_type != e1000_82547))
1075                 netdev->hw_features |= NETIF_F_TSO;
1076
1077         netdev->priv_flags |= IFF_SUPP_NOFCS;
1078
1079         netdev->features |= netdev->hw_features;
1080         netdev->hw_features |= NETIF_F_RXCSUM;
1081         netdev->hw_features |= NETIF_F_RXFCS;
```

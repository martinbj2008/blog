---
layout: post
title: "the timestamp in tcpdump Part1"
date: 2016-01-29 11:16:15 +0800
comments: true
categories: 
---

[tcpdump help](http://www.tcpdump.org/manpages/pcap-tstamp.7.html)
<!-- more -->

### case1: timestamp is get by ioctl (-j/J is not used)

Fox example tcpudmp direct dump to stdio, not write to file.

#### tcpdump call pcap_loop, while use print_packet as the callback.
```
 685 int
 686 main(int argc, char **argv)
 687 {
...
1475         if (WFileName) {
...
1510         } else {
1511                 type = pcap_datalink(pd);
1512                 printinfo = get_print_info(type);
1513                 callback = print_packet;
1514                 pcap_userdata = (u_char *)&printinfo;
1515         }
...
1568         do {
1569                 status = pcap_loop(pd, cnt, callback, pcap_userdata);
1570                 if (WFileName == NULL) {
...
1635         }
1636         while (ret != NULL);
1637
```
#### pcap_loop
```
 851 int
 852 pcap_loop(pcap_t *p, int cnt, pcap_handler callback, u_char *user)
 853 {
 854         register int n;
 855
 856         for (;;) {
 857                 if (p->rfile != NULL) {
 858                         /*
 859                          * 0 means EOF, so don't loop if we get 0.
 860                          */
 861                         n = pcap_offline_read(p, cnt, callback, user);
 862                 } else {
 863                         /*
 864                          * XXX keep reading until we get something
 865                          * (or an error occurs)
 866                          */
 867                         do {
 868                                 n = p->read_op(p, cnt, callback, user); <== here read_op is pcap_read_linux see "pcap_activate_linux"
 869                         } while (n == 0);
 870                 }
 871                 if (n <= 0)
 872                         return (n);
 873                 if (!PACKET_COUNT_IS_UNLIMITED(cnt)) {
 874                         cnt -= n;
 875                         if (cnt <= 0)
 876                                 return (0);
 877                 }
 878         }
 879 }
```

```
1229 static int
1230 pcap_activate_linux(pcap_t *handle)
1231 {
...
1261         handle->setnonblock_op = pcap_setnonblock_fd;
1262         handle->cleanup_op = pcap_cleanup_linux;
1263         handle->read_op = pcap_read_linux;
1264         handle->stats_op = pcap_stats_linux;
...
```

#### pcap_read_linux and pcap_read_packet

```
1412 pcap_read_linux(pcap_t *handle, int max_packets, pcap_handler callback, u_char *user)
1413 {
1414         /*
1415          * Currently, on Linux only one packet is delivered per read,
1416          * so we don't loop.
1417          */
1418         return pcap_read_packet(handle, callback, user);
1419 }
```

```
1469 static int
1470 pcap_read_packet(pcap_t *handle, pcap_handler callback, u_char *userdata)
1471 {
...
1493         struct pcap_pkthdr      pcap_header;
...
1737         /* get timestamp for this packet */
1738 #if defined(SIOCGSTAMPNS) && defined(SO_TIMESTAMPNS)
1739         if (handle->opt.tstamp_precision == PCAP_TSTAMP_PRECISION_NANO) {
1740                 if (ioctl(handle->fd, SIOCGSTAMPNS, &pcap_header.ts) == -1) {
1741                         snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
1742                                         "SIOCGSTAMPNS: %s", pcap_strerror(errno));
1743                         return PCAP_ERROR;
1744                 }
1745         } else
1746 #endif
1747         {
1748                 if (ioctl(handle->fd, SIOCGSTAMP, &pcap_header.ts) == -1) {
1749                         snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
1750                                         "SIOCGSTAMP: %s", pcap_strerror(errno));
1751                         return PCAP_ERROR;
1752                 }
1753         }
...
1804         /* Call the user supplied callback function */
1805         callback(userdata, &pcap_header, bp);   <=== here 'callback' is 'print_packet'
```

#### tcpdump use print_packet dump the timestamp
```
1929 static void
1930 print_packet(u_char *user, const struct pcap_pkthdr *h, const u_char *sp)
1931 {
...
1938         ts_print(&h->ts);
```
```
156 /*
157  * Print the timestamp
158  */
159 void
160 ts_print(register const struct timeval *tvp)
161 {
162         register int s;
163         struct tm *tm;
164         time_t Time;
165         static unsigned b_sec;
166         static unsigned b_usec;
167         int d_usec;
168         int d_sec;
169
170         switch (tflag) {
171
172         case 0: /* Default */
173                 s = (tvp->tv_sec + thiszone) % 86400;
174                 (void)printf("%s ", ts_format(s, tvp->tv_usec));
175                 break;
176
177         case 1: /* No time stamp */
178                 break;
179
180         case 2: /* Unix timeval style */
181                 (void)printf("%u.%06u ",
182                              (unsigned)tvp->tv_sec,
183                              (unsigned)tvp->tv_usec);
184                 break;
185
186         case 3: /* Microseconds since previous packet */
187         case 5: /* Microseconds since first packet */
188                 if (b_sec == 0) {
189                         /* init timestamp for first packet */
190                         b_usec = tvp->tv_usec;
191                         b_sec = tvp->tv_sec;
192                 }
193
194                 d_usec = tvp->tv_usec - b_usec;
195                 d_sec = tvp->tv_sec - b_sec;
196
197                 while (d_usec < 0) {
198                     d_usec += 1000000;
199                     d_sec--;
200                 }
201
202                 (void)printf("%s ", ts_format(d_sec, d_usec));
203
204                 if (tflag == 3) { /* set timestamp for last packet */
205                     b_sec = tvp->tv_sec;
206                     b_usec = tvp->tv_usec;
207                 }
208                 break;
209
210         case 4: /* Default + Date*/
211                 s = (tvp->tv_sec + thiszone) % 86400;
212                 Time = (tvp->tv_sec + thiszone) - s;
213                 tm = gmtime (&Time);
214                 if (!tm)
215                         printf("Date fail  ");
216                 else
217                         printf("%04d-%02d-%02d %s ",
218                                tm->tm_year+1900, tm->tm_mon+1, tm->tm_mday,
219                                ts_format(s, tvp->tv_usec));
220                 break;
221         }
222 }
```

#### libpcap
```
 623 int
 624 pcap_set_tstamp_type(pcap_t *p, int tstamp_type)
 625 {
 626         int i;
 627
 628         if (pcap_check_activated(p))
 629                 return (PCAP_ERROR_ACTIVATED);
 630
 631         /*
 632          * If p->tstamp_type_count is 0, we only support PCAP_TSTAMP_HOST;
 633          * the default time stamp type is PCAP_TSTAMP_HOST.
 634          */
 635         if (p->tstamp_type_count == 0) {
 636                 if (tstamp_type == PCAP_TSTAMP_HOST) {
 637                         p->opt.tstamp_type = tstamp_type;
 638                         return (0);
 639                 }
 640         } else {
 641                 /*
 642                  * Check whether we claim to support this type of time stamp.
 643                  */
 644                 for (i = 0; i < p->tstamp_type_count; i++) {
 645                         if (p->tstamp_type_list[i] == tstamp_type) {
 646                                 /*
 647                                  * Yes.
 648                                  */
 649                                 p->opt.tstamp_type = tstamp_type;
 650                                 return (0);
 651                         }
 652                 }
 653         }
 654
 655         /*
 656          * We don't support this type of time stamp.
 657          */
 658         return (PCAP_WARNING_TSTAMP_TYPE_NOTSUP);
 659 }
```


```
3801         /*
3802          * If we were told to do so, ask the kernel and the driver
3803          * to use hardware timestamps.
3804          *
3805          * Hardware timestamps are only supported with mmapped
3806          * captures.
3807          */
3808         if (handle->opt.tstamp_type == PCAP_TSTAMP_ADAPTER ||
3809             handle->opt.tstamp_type == PCAP_TSTAMP_ADAPTER_UNSYNCED) {
3810                 struct hwtstamp_config hwconfig;
3811                 struct ifreq ifr;
3812                 int timesource;
3813
3814                 /*
3815                  * Ask for hardware time stamps on all packets,
3816                  * including transmitted packets.
3817                  */
3818                 memset(&hwconfig, 0, sizeof(hwconfig));
3819                 hwconfig.tx_type = HWTSTAMP_TX_ON;
3820                 hwconfig.rx_filter = HWTSTAMP_FILTER_ALL;
3821
3822                 memset(&ifr, 0, sizeof(ifr));
3823                 strlcpy(ifr.ifr_name, handle->opt.source, sizeof(ifr.ifr_name));
3824                 ifr.ifr_data = (void *)&hwconfig;
3825
3826                 if (ioctl(handle->fd, SIOCSHWTSTAMP, &ifr) < 0) {
3827                         switch (errno) {
3828
3829                         case EPERM:
3830                                 /*
3831                                  * Treat this as an error, as the
3832                                  * user should try to run this
3833                                  * with the appropriate privileges -
3834                                  * and, if they can't, shouldn't
3835                                  * try requesting hardware time stamps.
3836                                  */
3837                                 *status = PCAP_ERROR_PERM_DENIED;
3838                                 return -1;
3839
3840                         case EOPNOTSUPP:
3841                                 /*
3842                                  * Treat this as a warning, as the
3843                                  * only way to fix the warning is to
3844                                  * get an adapter that supports hardware
3845                                  * time stamps.  We'll just fall back
3846                                  * on the standard host time stamps.
3847                                  */
3848                                 *status = PCAP_WARNING_TSTAMP_TYPE_NOTSUP;
3849                                 break;
3850
3851                         default:
3852                                 snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
3853                                         "SIOCSHWTSTAMP failed: %s",
3854                                         pcap_strerror(errno));
3855                                 *status = PCAP_ERROR;
3856                                 return -1;
3857                         }
3858                 } else {
3859                         /*
3860                          * Well, that worked.  Now specify the type of
3861                          * hardware time stamp we want for this
3862                          * socket.
3863                          */
3864                         if (handle->opt.tstamp_type == PCAP_TSTAMP_ADAPTER) {
3865                                 /*
3866                                  * Hardware timestamp, synchronized
3867                                  * with the system clock.
3868                                  */
3869                                 timesource = SOF_TIMESTAMPING_SYS_HARDWARE;
3870                         } else {
3871                                 /*
3872                                  * PCAP_TSTAMP_ADAPTER_UNSYNCED - hardware
3873                                  * timestamp, not synchronized with the
3874                                  * system clock.
3875                                  */
3876                                 timesource = SOF_TIMESTAMPING_RAW_HARDWARE;
3877                         }
3878                         if (setsockopt(handle->fd, SOL_PACKET, PACKET_TIMESTAMP,
3879                                 (void *)&timesource, sizeof(timesource))) {
3880                                 snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
3881                                         "can't set PACKET_TIMESTAMP: %s",
3882                                         pcap_strerror(errno));
3883                                 *status = PCAP_ERROR;
3884                                 return -1;
3885                         }
3886                 }
3887         }
```

####  af_packet of kernel 
1. af_packet
```
3505 static int
3506 packet_setsockopt(struct socket *sock, int level, int optname, char __user *optval, unsigned int optlen)
3507 {
...
3660         case PACKET_TIMESTAMP:
3661         {
3662                 int val;
3663
3664                 if (optlen != sizeof(val))
3665                         return -EINVAL;
3666                 if (copy_from_user(&val, optval, sizeof(val)))
3667                         return -EFAULT;
3668
3669                 po->tp_tstamp = val;
3670                 return 0;
3671         }
```

```
2087 static int tpacket_rcv(struct sk_buff *skb, struct net_device *dev,
2088                        struct packet_type *pt, struct net_device *orig_dev)
2089 {
...
2209         if (!(ts_status = tpacket_get_timestamp(skb, &ts, po->tp_tstamp)))
2210                 getnstimeofday(&ts);
2211
2212         status |= ts_status;
2213
2214         switch (po->tp_version) {
2215         case TPACKET_V1:
2216                 h.h1->tp_len = skb->len;
2217                 h.h1->tp_snaplen = snaplen;
2218                 h.h1->tp_mac = macoff;
2219                 h.h1->tp_net = netoff;
2220                 h.h1->tp_sec = ts.tv_sec;
2221                 h.h1->tp_usec = ts.tv_nsec / NSEC_PER_USEC;
2222                 hdrlen = sizeof(*h.h1);
2223                 break;
2224         case TPACKET_V2:
2225                 h.h2->tp_len = skb->len;
2226                 h.h2->tp_snaplen = snaplen;
2227                 h.h2->tp_mac = macoff;
2228                 h.h2->tp_net = netoff;
2229                 h.h2->tp_sec = ts.tv_sec;
2230                 h.h2->tp_nsec = ts.tv_nsec;
2231                 if (skb_vlan_tag_present(skb)) {
2232                         h.h2->tp_vlan_tci = skb_vlan_tag_get(skb);
2233                         h.h2->tp_vlan_tpid = ntohs(skb->vlan_proto);
2234                         status |= TP_STATUS_VLAN_VALID | TP_STATUS_VLAN_TPID_VALID;
2235                 } else {
2236                         h.h2->tp_vlan_tci = 0;
2237                         h.h2->tp_vlan_tpid = 0;
2238                 }
2239                 memset(h.h2->tp_padding, 0, sizeof(h.h2->tp_padding));
```

```
 442 static __u32 tpacket_get_timestamp(struct sk_buff *skb, struct timespec *ts,
 443                                    unsigned int flags)
 444 {
 445         struct skb_shared_hwtstamps *shhwtstamps = skb_hwtstamps(skb);
 446
 447         if (shhwtstamps &&
 448             (flags & SOF_TIMESTAMPING_RAW_HARDWARE) &&
 449             ktime_to_timespec_cond(shhwtstamps->hwtstamp, ts))
 450                 return TP_STATUS_TS_RAW_HARDWARE;
 451
 452         if (ktime_to_timespec_cond(skb->tstamp, ts))
 453                 return TP_STATUS_TS_SOFTWARE;
 454
 455         return 0;
 456 }
 457
```

2. ixgbe driver
```
 315 static void ixgbe_ptp_convert_to_hwtstamp(struct ixgbe_adapter *adapter,
 316                                           struct skb_shared_hwtstamps *hwtstamp,
 317                                           u64 timestamp)
 318 {
 319         unsigned long flags;
 320         struct timespec64 systime;
 321         u64 ns;
 322
 323         memset(hwtstamp, 0, sizeof(*hwtstamp));
 324
 325         switch (adapter->hw.mac.type) {
 326         /* X550 and later hardware supposedly represent time using a seconds
 327          * and nanoseconds counter, instead of raw 64bits nanoseconds. We need
 328          * to convert the timestamp into cycles before it can be fed to the
 329          * cyclecounter. We need an actual cyclecounter because some revisions
 330          * of hardware run at a higher frequency and thus the counter does
 331          * not represent seconds/nanoseconds. Instead it can be thought of as
 332          * cycles and billions of cycles.
 333          */
 334         case ixgbe_mac_X550:
 335         case ixgbe_mac_X550EM_x:
 336                 /* Upper 32 bits represent billions of cycles, lower 32 bits
 337                  * represent cycles. However, we use timespec64_to_ns for the
 338                  * correct math even though the units haven't been corrected
 339                  * yet.
 340                  */
 341                 systime.tv_sec = timestamp >> 32;
 342                 systime.tv_nsec = timestamp & 0xFFFFFFFF;
 343
 344                 timestamp = timespec64_to_ns(&systime);
 345                 break;
 346         default:
 347                 break;
 348         }
 349
 350         spin_lock_irqsave(&adapter->tmreg_lock, flags);
 351         ns = timecounter_cyc2time(&adapter->hw_tc, timestamp);
 352         spin_unlock_irqrestore(&adapter->tmreg_lock, flags);
 353
 354         hwtstamp->hwtstamp = ns_to_ktime(ns);
 355 }
```


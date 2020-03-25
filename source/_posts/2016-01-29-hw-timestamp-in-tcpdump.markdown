---
layout: post
title: "hw timestamp in tcpdump"
date: 2016-01-29 14:15:26 +0800
comments: true
categories: 
---

### PCAP_TSTAMP_ADAPTER  and PCAP_TSTAMP_ADAPTER_UNSYNCED
[tcpdump help](http://www.tcpdump.org/manpages/pcap-tstamp.7.html)


#### tcpudmp use -j option
tcpdump main function

<!-- more -->

```
1296 #ifdef HAVE_PCAP_SET_TSTAMP_TYPE
1297                 if (jflag != -1) {
1298                         status = pcap_set_tstamp_type(pd, jflag);
1299                         if (status < 0)
1300                                 error("%s: Can't set time stamp type: %s",
1301                                     device, pcap_statustostr(status));
1302                 }
1303 #endif
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

pcap_activate(pd);
```
 727 int
 728 pcap_activate(pcap_t *p)
 729 {
 730         int status;
 731
 732         /*
 733          * Catch attempts to re-activate an already-activated
 734          * pcap_t; this should, for example, catch code that
 735          * calls pcap_open_live() followed by pcap_activate(),
 736          * as some code that showed up in a Stack Exchange
 737          * question did.
 738          */
 739         if (pcap_check_activated(p))
 740                 return (PCAP_ERROR_ACTIVATED);
 741         status = p->activate_op(p); <== activate_op 相当于 pcap_activate_linux，见pcap_create_interface
 742         if (status >= 0)
 743                 p->activated = 1;
 744         else {
...
```

```
 412 pcap_t *
 413 pcap_create_interface(const char *device, char *ebuf)
 414 {
 415         pcap_t *handle;
 416
 417         handle = pcap_create_common(device, ebuf, sizeof (struct pcap_linux));
 418         if (handle == NULL)
 419                 return NULL;
 420
 421         handle->activate_op = pcap_activate_linux;
 422         handle->can_set_rfmon_op = pcap_can_set_rfmon_linux;
```

```
1229 static int
1230 pcap_activate_linux(pcap_t *handle)
1231 {
1232         struct pcap_linux *handlep = handle->priv;
...
1299         /*
1300          * Current Linux kernels use the protocol family PF_PACKET to
1301          * allow direct access to all packets on the network while
1302          * older kernels had a special socket type SOCK_PACKET to
1303          * implement this feature.
1304          * While this old implementation is kind of obsolete we need
1305          * to be compatible with older kernels for a while so we are
1306          * trying both methods with the newer method preferred.
1307          */
1308         ret = activate_new(handle);
1309         if (ret < 0) {
1310                 /*
1311                  * Fatal error with the new way; just fail.
1312                  * ret has the error return; if it's PCAP_ERROR,
1313                  * handle->errbuf has been set appropriately.
1314                  */
1315                 status = ret;
1316                 goto fail;
1317         }
1318         if (ret == 1) {
1319                 /*
1320                  * Success.
1321                  * Try to use memory-mapped access.
1322                  */
1323                 switch (activate_mmap(handle, &status)) {
1324
```

```

3387 /*
3388  * Attempt to activate with memory-mapped access.
3389  *
3390  * On success, returns 1, and sets *status to 0 if there are no warnings
3391  * or to a PCAP_WARNING_ code if there is a warning.
3392  *
3393  * On failure due to lack of support for memory-mapped capture, returns
3394  * 0.
3395  *
3396  * On error, returns -1, and sets *status to the appropriate error code;
3397  * if that is PCAP_ERROR, sets handle->errbuf to the appropriate message.
3398  */
3399 static int
3400 activate_mmap(pcap_t *handle, int *status)
3401 {
...
3428         ret = create_ring(handle, status);
```

```
3611 static int
3612 create_ring(pcap_t *handle, int *status)
...
3787         /*
3788          * PACKET_TIMESTAMP was added after linux/net_tstamp.h was,
3789          * so we check for PACKET_TIMESTAMP.  We check for
3790          * linux/net_tstamp.h just in case a system somehow has
3791          * PACKET_TIMESTAMP but not linux/net_tstamp.h; that might
3792          * be unnecessary.
3793          *
3794          * SIOCSHWTSTAMP was introduced in the patch that introduced
3795          * linux/net_tstamp.h, so we don't bother checking whether
3796          * SIOCSHWTSTAMP is defined (if your Linux system has
3797          * linux/net_tstamp.h but doesn't define SIOCSHWTSTAMP, your
3798          * Linux system is badly broken).
3799          */
3800 #if defined(HAVE_LINUX_NET_TSTAMP_H) && defined(PACKET_TIMESTAMP)
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
3878                         if (setsockopt(handle->fd, SOL_PACKET, PACKET_TIMESTAMP,   <=======
3879                                 (void *)&timesource, sizeof(timesource))) {
3880                                 snprintf(handle->errbuf, PCAP_ERRBUF_SIZE,
3881                                         "can't set PACKET_TIMESTAMP: %s",
3882                                         pcap_strerror(errno));
3883                                 *status = PCAP_ERROR;
3884                                 return -1;
3885                         }
3886                 }
3887         }
3888 #endif /* HAVE_LINUX_NET_TSTAMP_H && PACKET_TIMESTAMP */
```

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

packet recv
########################################################################################
### case2: timestamp in tcpdump while -j/J used


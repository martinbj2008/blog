---
layout: post
title: "Netlink in kernel"
date: 2013-02-16 00:00
comments: true
categories: [netlink]
tags: [kernel, netlink]
---

#netlink socket framework.

##netlink socket proto
  netlink socket is a kind of socket. There are many proto of netlink. There may have many groups under each proto. See example in following.

Every netlink socket is indicated by `<net, proto, pid>`.

  1. net: the net namespace.
  2. proto: netlink proto.
  3. pid:

<!-- more -->

```c
  7 #define NETLINK_ROUTE           0       /* Routing/device hook                          */
  8 #define NETLINK_UNUSED          1       /* Unused number                                */
  9 #define NETLINK_USERSOCK        2       /* Reserved for user mode socket protocols      */
 10 #define NETLINK_FIREWALL        3       /* Firewalling hook                             */
 11 #define NETLINK_SOCK_DIAG       4       /* socket monitoring                            */
 12 #define NETLINK_NFLOG           5       /* netfilter/iptables ULOG */
 13 #define NETLINK_XFRM            6       /* ipsec */
 14 #define NETLINK_SELINUX         7       /* SELinux event notifications */
 15 #define NETLINK_ISCSI           8       /* Open-iSCSI */
 16 #define NETLINK_AUDIT           9       /* auditing */
 17 #define NETLINK_FIB_LOOKUP      10
 18 #define NETLINK_CONNECTOR       11
 19 #define NETLINK_NETFILTER       12      /* netfilter subsystem */
 20 #define NETLINK_IP6_FW          13
 21 #define NETLINK_DNRTMSG         14      /* DECnet routing messages */
 22 #define NETLINK_KOBJECT_UEVENT  15      /* Kernel messages to userspace */
 23 #define NETLINK_GENERIC         16
 24 /* leave room for NETLINK_DM (DM Events) */
 25 #define NETLINK_SCSITRANSPORT   18      /* SCSI Transports */
 26 #define NETLINK_ECRYPTFS        19
 27 #define NETLINK_RDMA            20
 28 #define NETLINK_CRYPTO          21      /* Crypto layer */
 29
 30 #define NETLINK_INET_DIAG       NETLINK_SOCK_DIAG
 31
 32 #define MAX_LINKS 32
```
Each proto of netlink has many groups, for example xfrm netlink:
```c
462 enum xfrm_nlgroups {
463         XFRMNLGRP_NONE,
464 #define XFRMNLGRP_NONE          XFRMNLGRP_NONE
465         XFRMNLGRP_ACQUIRE,
466 #define XFRMNLGRP_ACQUIRE       XFRMNLGRP_ACQUIRE
467         XFRMNLGRP_EXPIRE,
468 #define XFRMNLGRP_EXPIRE        XFRMNLGRP_EXPIRE
469         XFRMNLGRP_SA,
470 #define XFRMNLGRP_SA            XFRMNLGRP_SA
471         XFRMNLGRP_POLICY,
472 #define XFRMNLGRP_POLICY        XFRMNLGRP_POLICY
473         XFRMNLGRP_AEVENTS,
474 #define XFRMNLGRP_AEVENTS       XFRMNLGRP_AEVENTS
475         XFRMNLGRP_REPORT,
476 #define XFRMNLGRP_REPORT        XFRMNLGRP_REPORT
477         XFRMNLGRP_MIGRATE,
478 #define XFRMNLGRP_MIGRATE       XFRMNLGRP_MIGRATE
479         XFRMNLGRP_MAPPING,
480 #define XFRMNLGRP_MAPPING       XFRMNLGRP_MAPPING
481         __XFRMNLGRP_MAX
482 };
483 #define XFRMNLGRP_MAX   (__XFRMNLGRP_MAX - 1)
```
all the supported protos are stored in the array `nl_table`.

`static struct netlink_table *nl_table`;

each type is stored in a struct netlink_table
```c
119 struct netlink_table {
120         struct nl_pid_hash hash;          /* store all the netlink socket with hash list */
121         struct hlist_head mc_list;            /* store all the netlink socket listen broadcast netlink message */
122         struct listeners __rcu *listeners;      /* store with a bit(1) for each netlink broadcast type, which is listened by one or more netlink sockets in mc_list(line 121) */
123         unsigned int nl_nonroot;
124         unsigned int groups;            /* the counts of bits in listeners(line 122)*/
125         struct mutex *cb_mutex;
126         struct module *module;
127         int registered;             /* avoid repeated register. */
128 };
```
In initialization, each type netlink call netlink_kernel_create, which will prepare two parts:

1. Register in a netlink_table of nl_table,
2. Create a netlink kernel socket for the registered proto.

##Example: xfrm netlink
  Register a netlink type with NETLINK_XFRM, which has XFRMNLGRP_MAX groups. and all the netlink message will be processed by xfrm_netlink_rcv.
```c
2603 static int __net_init xfrm_user_net_init(struct net *net)
2604 {
2605         struct sock *nlsk;
2606
2607         nlsk = netlink_kernel_create(net, NETLINK_XFRM, XFRMNLGRP_MAX,
2608                                      xfrm_netlink_rcv, NULL, THIS_MODULE);
2609         if (nlsk == NULL)
2610                 return -ENOMEM;
2611         rcu_assign_pointer(net->xfrm.nlsk, nlsk);
2612         return 0;
2613 }
```
###Data struct && method

####struct netlink_sock

```c
67 struct netlink_sock {
68         /* struct sock has to be the first member of netlink_sock */
69         struct sock             sk;
70         u32                     pid;
71         u32                     dst_pid;
72         u32                     dst_group;
73         u32                     flags;
74         u32                     subscriptions;
75         u32                     ngroups;
76         unsigned long           *groups;
77         unsigned long           state;
78         wait_queue_head_t       wait;
79         struct netlink_callback *cb;
80         struct mutex            *cb_mutex;
81         struct mutex            cb_def_mutex;
82         void                    (*netlink_rcv)(struct sk_buff *skb);
83         struct module           *module;
84 };
```

Every netlink socket has a pid. The pid of netlink kernel sockets is 0. All the the netlink message need kernel process, will be sent to them. (net, pid, protocol) will be used to locate the destination of a netlink message.

####netlink_create
```c
 430 static int netlink_create(struct net *net, struct socket *sock, int protocol,
 431                           int kern)
 432 {
 433         struct module *module = NULL;
 434         struct mutex *cb_mutex;
 435         struct netlink_sock *nlk;
 436         int err = 0;
 437
 438         sock->state = SS_UNCONNECTED;
 439
 440         if (sock->type != SOCK_RAW && sock->type != SOCK_DGRAM)
 441                 return -ESOCKTNOSUPPORT;
 442
 443         if (protocol < 0 || protocol >= MAX_LINKS)
 444                 return -EPROTONOSUPPORT;
 445
 446         netlink_lock_table();
 447 #ifdef CONFIG_MODULES
 448         if (!nl_table[protocol].registered) {
 449                 netlink_unlock_table();
 450                 request_module("net-pf-%d-proto-%d", PF_NETLINK, protocol);
 451                 netlink_lock_table();
 452         }
 453 #endif
 454         if (nl_table[protocol].registered &&
 455             try_module_get(nl_table[protocol].module))
 456                 module = nl_table[protocol].module;
 457         else
 458                 err = -EPROTONOSUPPORT;
 459         cb_mutex = nl_table[protocol].cb_mutex;
 460         netlink_unlock_table();
 461
 462         if (err < 0)
 463                 goto out;
 464
 465         err = __netlink_create(net, sock, cb_mutex, protocol);
 466         if (err < 0)
 467                 goto out_module;
 468
 469         local_bh_disable();
 470         sock_prot_inuse_add(net, &netlink_proto, 1);
 471         local_bh_enable();
 472
 473         nlk = nlk_sk(sock->sk);
 474         nlk->module = module;
 475 out:
 476         return err;
 477
 478 out_module:
 479         module_put(module);
 480         goto out;
 481 }
```

```c
402 static int __netlink_create(struct net *net, struct socket *sock,
403                             struct mutex *cb_mutex, int protocol)
404 {
405         struct sock *sk;
406         struct netlink_sock *nlk;
407
408         sock->ops = &netlink_ops;
409
410         sk = sk_alloc(net, PF_NETLINK, GFP_KERNEL, &netlink_proto);
411         if (!sk)
412                 return -ENOMEM;
413
414         sock_init_data(sock, sk);
415
416         nlk = nlk_sk(sk);
417         if (cb_mutex)
418                 nlk->cb_mutex = cb_mutex;
419         else {
420                 nlk->cb_mutex = &nlk->cb_def_mutex;
421                 mutex_init(nlk->cb_mutex);
422         }
423         init_waitqueue_head(&nlk->wait);
424
425         sk->sk_destruct = netlink_sock_destruct;
426         sk->sk_protocol = protocol;
427         return 0;
428 }
```

####netlink_sendmsg
```
netlink_sendmsg —–> netlink_unicastg —–> nlk->netlink_rcv,
```
Create a skb and init `NETLINK_CB(skb)`, Copy the message data with memcpy_fromiovec.

If its destination is a group, then broadcast it with netlink_broadcast, or it is a unicast message, send it by netlink_unicast.

`nlk->netlink_rcv` is set in `netlink_create` for xfrm netlink message, it is`xfrm_netlink_rcv`.
```c
1313 static int netlink_sendmsg(struct kiocb *kiocb, struct socket *sock,
1314                            struct msghdr *msg, size_t len)
1315 {
1316         struct sock_iocb *siocb = kiocb_to_siocb(kiocb);
1317         struct sock *sk = sock->sk;
1318         struct netlink_sock *nlk = nlk_sk(sk);
1319         struct sockaddr_nl *addr = msg->msg_name;
1320         u32 dst_pid;
1321         u32 dst_group;
1322         struct sk_buff *skb;
1323         int err;
1324         struct scm_cookie scm;
1325
1326         if (msg->msg_flags&MSG_OOB)
1327                 return -EOPNOTSUPP;
1328
1329         if (NULL == siocb->scm)
1330                 siocb->scm = &scm;
1331
1332         err = scm_send(sock, msg, siocb->scm, true);
1333         if (err < 0)
1334                 return err;
1335
1336         if (msg->msg_namelen) {
1337                 err = -EINVAL;
1338                 if (addr->nl_family != AF_NETLINK)
1339                         goto out;
1340                 dst_pid = addr->nl_pid;
1341                 dst_group = ffs(addr->nl_groups);
1342                 err =  -EPERM;
1343                 if ((dst_group || dst_pid) &&
1344                     !netlink_capable(sock, NL_NONROOT_SEND))
1345                         goto out;
1346         } else {
1347                 dst_pid = nlk->dst_pid;
1348                 dst_group = nlk->dst_group;
1349         }
1350
1351         if (!nlk->pid) {
1352                 err = netlink_autobind(sock);
1353                 if (err)
1354                         goto out;
1355         }
1356
1357         err = -EMSGSIZE;
1358         if (len > sk->sk_sndbuf - 32)
1359                 goto out;
1360         err = -ENOBUFS;
1361         skb = alloc_skb(len, GFP_KERNEL);
1362         if (skb == NULL)
1363                 goto out;
1364
1365         NETLINK_CB(skb).pid     = nlk->pid;
1366         NETLINK_CB(skb).dst_group = dst_group;
1367         memcpy(NETLINK_CREDS(skb), &siocb->scm->creds, sizeof(struct ucred));
1368
1369         err = -EFAULT;
1370         if (memcpy_fromiovec(skb_put(skb, len), msg->msg_iov, len)) {
1371                 kfree_skb(skb);
1372                 goto out;
1373         }
1374
1375         err = security_netlink_send(sk, skb);
1376         if (err) {
1377                 kfree_skb(skb);
1378                 goto out;
1379         }
1380
1381         if (dst_group) {
1382                 atomic_inc(&skb->users);
1383                 netlink_broadcast(sk, skb, dst_pid, dst_group, GFP_KERNEL);
1384         }
1385         err = netlink_unicast(sk, skb, dst_pid, msg->msg_flags&MSG_DONTWAIT);
1386
1387 out:
1388         scm_destroy(siocb->scm);
1389         return err;
1390 }
```
###netlink_unicast
  find the dest netlink socket by its pid.
if dst socket is a kernl socket, call netlink_unicast_kernel. for example : in the case, dump xfrm sa. else put the skb into the dest socket recv queue.

```c
891 int netlink_unicast(struct sock ssk, struct sk_buff skb, 
892 u32 pid, int nonblock)
893 { 
894 struct sock *sk; 
895 int err; 
896 long timeo; 
897 
898 skb = netlink_trim(skb, gfp_any()); 
899 
900 timeo = sock_sndtimeo(ssk, nonblock); 
901 retry: 
902 sk = netlink_getsockbypid(ssk, pid); 
903 if (IS_ERR(sk)) { 
904 kfree_skb(skb); 
905 return PTR_ERR(sk); 
906 } 
907 if (netlink_is_kernel(sk)) 
908 return netlink_unicast_kernel(sk, skb); 
909 
910 if (sk_filter(sk, skb)) { 
911 err = skb->len; 
912 kfree_skb(skb); 
913 sock_put(sk); 
914 return err; 
915 } 
916 
917 err = netlink_attachskb(sk, skb, &timeo, ssk); 
918 if (err == 1) 
919 goto retry; 
920 if (err) 
921 return err; 
922 
923 return netlink_sendskb(sk, skb); 
924 } 
925 EXPORT_SYMBOL(netlink_unicast);
```

###broadcast message
```c
 905 int netlink_unicast(struct sock *ssk, struct sk_buff *skb,
 906                     u32 pid, int nonblock)
 907 {
 908         struct sock *sk;
 909         int err;
 910         long timeo;
 911
 912         skb = netlink_trim(skb, gfp_any());
 913
 914         timeo = sock_sndtimeo(ssk, nonblock);
 915 retry:
 916         sk = netlink_getsockbypid(ssk, pid);
 917         if (IS_ERR(sk)) {
 918                 kfree_skb(skb);
 919                 return PTR_ERR(sk);
 920         }
 921         if (netlink_is_kernel(sk))
 922                 return netlink_unicast_kernel(sk, skb);
 923
 924         if (sk_filter(sk, skb)) {
 925                 err = skb->len;
 926                 kfree_skb(skb);
 927                 sock_put(sk);
 928                 return err;
 929         }
 930
 931         err = netlink_attachskb(sk, skb, &timeo, ssk);
 932         if (err == 1)
 933                 goto retry;
 934         if (err)
 935                 return err;
 936
 937         return netlink_sendskb(sk, skb);
 938 }
```
```c
 889 static int netlink_unicast_kernel(struct sock *sk, struct sk_buff *skb)
 890 {
 891         int ret;
 892         struct netlink_sock *nlk = nlk_sk(sk);
 893
 894         ret = -ECONNREFUSED;
 895         if (nlk->netlink_rcv != NULL) {
 896                 ret = skb->len;
 897                 skb_set_owner_r(skb, sk);
 898                 nlk->netlink_rcv(skb);
 899         }
 900         kfree_skb(skb);
 901         sock_put(sk);
 902         return ret;
 903 }
```
```c
  86 struct listeners {
  87         struct rcu_head         rcu;
  88         unsigned long           masks[0];
  89 };
```

```c
106 struct nl_pid_hash {
 107         struct hlist_head *table;
 108         unsigned long rehash_time;
 109
 110         unsigned int mask;
 111         unsigned int shift;
 112
 113         unsigned int entries;
 114         unsigned int max_shift;
 115
 116         u32 rnd;
 117 };
```

```c
  55 struct rtnl_link {
  56         rtnl_doit_func          doit;
  57         rtnl_dumpit_func        dumpit;
  58         rtnl_calcit_func        calcit;
  59 };
```
```
101 static struct rtnl_link *rtnl_msg_handlers[RTNL_FAMILY_MAX + 1];
```

###pernet ops
```
2124 static struct pernet_operations __net_initdata netlink_net_ops = {
2125         .init = netlink_net_init,
2126         .exit = netlink_net_exit,
2127 };
```
###socket related ops

```
 396 static struct proto netlink_proto = {
 397         .name     = "NETLINK",
 398         .owner    = THIS_MODULE,
 399         .obj_size = sizeof(struct netlink_sock),
 400 };
```
```
2061 static const struct proto_ops netlink_ops = {
2062         .family =       PF_NETLINK,
2063         .owner =        THIS_MODULE,
2064         .release =      netlink_release,
2065         .bind =         netlink_bind,
2066         .connect =      netlink_connect,
2067         .socketpair =   sock_no_socketpair,
2068         .accept =       sock_no_accept,
2069         .getname =      netlink_getname,
2070         .poll =         datagram_poll,
2071         .ioctl =        sock_no_ioctl,
2072         .listen =       sock_no_listen,
2073         .shutdown =     sock_no_shutdown,
2074         .setsockopt =   netlink_setsockopt,
2075         .getsockopt =   netlink_getsockopt,
2076         .sendmsg =      netlink_sendmsg,
2077         .recvmsg =      netlink_recvmsg,
2078         .mmap =         sock_no_mmap,
2079         .sendpage =     sock_no_sendpage,
2080 };
```
```
2082 static const struct net_proto_family netlink_family_ops = {
2083         .family = PF_NETLINK,
2084         .create = netlink_create,
2085         .owner  = THIS_MODULE,  /* for consistency 8) */
2086 };
```
###Initialization

```c
2129 static int __init netlink_proto_init(void)
2130 {
2131         struct sk_buff *dummy_skb;
2132         int i;
2133         unsigned long limit;
2134         unsigned int order;
2135         int err = proto_register(&netlink_proto, 0);
2136
2137         if (err != 0)
2138                 goto out;
2139
2140         BUILD_BUG_ON(sizeof(struct netlink_skb_parms) > sizeof(dummy_skb->cb));
2141
2142         nl_table = kcalloc(MAX_LINKS, sizeof(*nl_table), GFP_KERNEL);
2143         if (!nl_table)
2144                 goto panic;
2145
2146         if (totalram_pages >= (128 * 1024))
2147                 limit = totalram_pages >> (21 - PAGE_SHIFT);
2148         else
2149                 limit = totalram_pages >> (23 - PAGE_SHIFT);
2150
2151         order = get_bitmask_order(limit) - 1 + PAGE_SHIFT;
2152         limit = (1UL << order) / sizeof(struct hlist_head);
2153         order = get_bitmask_order(min(limit, (unsigned long)UINT_MAX)) - 1;
2154
2155         for (i = 0; i < MAX_LINKS; i++) {
2156                 struct nl_pid_hash *hash = &nl_table[i].hash;
2157
2158                 hash->table = nl_pid_hash_zalloc(1 * sizeof(*hash->table));
2159                 if (!hash->table) {
2160                         while (i-- > 0)
2161                                 nl_pid_hash_free(nl_table[i].hash.table,
2162                                                  1 * sizeof(*hash->table));
2163                         kfree(nl_table);
2164                         goto panic;
2165                 }
2166                 hash->max_shift = order;
2167                 hash->shift = 0;
2168                 hash->mask = 0;
2169                 hash->rehash_time = jiffies;
2170         }
2171
2172         netlink_add_usersock_entry();
2173
2174         sock_register(&netlink_family_ops);
2175         register_pernet_subsys(&netlink_net_ops);
2176         /* The netlink device handler may be needed early. */
2177         rtnetlink_init();
2178 out:
2179         return err;
2180 panic:
2181         panic("netlink_init: Cannot allocate nl_table\n");
2182 }
2183
2184 core_initcall(netlink_proto_init);
```

---
layout: post
title: "Netlink in kernel(continue)"
date: 2013-03-15 00:00
comments: true
categories: [netlink]
tags: [kernel, netlink]
---

##netlink介绍
  netlink是一种用于内核和用户空间进行数据交互的socket。 关于netlink的具体介绍，google给出更好的解释。
  netlink是socket的一种，其的family号是PF_NETLINK, netlink包括很多种proto，并且用户可以根据自己的需要进行扩展。
每个netlinksocket都有一个pid，该pid在所属proto下是唯一的。 在netlink消息
传递是，pid常被用来标识目的地socket。 

Every netlink socket is indicated by (net, proto, pid). 

  1. net: the net namespace. 
  2. proto: netlink proto. 
  3. pid:

<!-- more -->

netlink的消息可以分为两种类型： 单播(unicast)和组播(broadcast). 

  1. 单播：目的socket是固定的。 
  2. 组播：目的socket可以是一个，或者多个，甚至还可能是没有。 组播消息为被发送到每一个监听对应group的socket的接受队列上。

netlink的使用跟普通socket基本相同。 “create/bind send/recv”

每个netlink socket在创建时候必须指明 family 和proto。 family当然是netlink， proto可以个netlink支持的任意一个。

每个proto下都可以有包含很多个组播组（group）。 如xfrm proto下面包含

每个socket的创建是都可以指定对一个或者几个group感兴趣， 那么内核就自动将相关的group上的组播消息组播到该socket上。 该socket通过recv函数接受这些组播消息。

注：每个netlinksocket在创建的时候必须指明所属的proto。 因此每个netlink socket只能监听其所在proto下的group组播消息。

example： dump出内核里所有的interface， 那么我创建一个netlink socket， 然后调用send/rcv让内核吧所有的interface信息输出。

如果我们还要求： 监控interface状态改变， 比如 当”interface down/up” 我们第一时间知道interface的新状态。 那么我们在创建sock的时候需指明要监听groupXX. group可以在创建socket时候指明， 也可以在socket创建后，通过set sockopt来更改。 当然也可在程序运行过程中通过sockopt更改。 这个group信息会保存在netlink sock的group里， 同时也会更新对应proto下的group信息。

见示例代码。xxx todo.

netlink_table

netlink支持的所有proto都保存在一个数组nl_table中.
```c
static struct netlink_table *nl_table;
```

也就是每个proto都对应一个`struct netlink_table`.

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

针对特定的一个proto: 

  1. hash: 所有的proto类型的netlink socket 都放到这个hash链表里。 
  2. mc_list: hash链表。记录希望监听组播消息的netlink socket.
  3. listeners: 记录哪些group被监听, 一个group占1位.
  4. nl_noroot: todo! 
  5. groups: 被监听的group的总个数 
  6. cb_mutex: callback 函数用到的锁 
  7. module: 在lsm模式下使用（不严格）。 
  8. register: 是否被注册（占用）。

同一个proto下的所有netlink socket都会挂载到 hash链上， 这些socket中凡是对希望监听group组播消息的， 都还会被挂载到mc_list上。 不论是要监听多少个group，每个sock都只作为mc_list一个节点，不会重复。

当有组播消息产生时，kernel首先查看netlink_table->groups是否为0， 如果为0， 那么没有任何一个group没监听。显然，这种情况下，kernel啥也不用做。 直接退出。

如果netlink_table->groups不为0，那么kernel会遍历mc_list， 取出每个netlink socket查看当前组播消息的类型是否是 在该socket所监听的goups里，如果是那么发送当前消息到该socket的接受队列里。 否则啥也不做，继续遍历mc_list里下一个netlink socket.

如果要投递单播消息那么直接根据 netlink sock的pid， 在hash链里找到对应的socket并发送消息副本(skb clone or copy)到对应的sk接收队列 同时激发data_ready.

[netlink_table的读写保护] (http://martinbj2008.github.com/kernel/2013/03/11/netlink-grab/)

综述

每个netlink socket 被创建的时候，proto已经被指定。 1. 保存了自己的groups，用来表明自己对莫个(些）group感兴趣。 2. 挂在自己的socket的到netlink_table->hash下。 3. 根据其是否对组播感兴趣（sock->groups == 0 ?) ，决定是否挂到netlink_table->mc_list下 并根据需要更行netlink_table下的listeners的标志位和groups，

每个proto在初始化时都: 1. 创建一个内核态netlink socket。(pid 为0) 2. 注册到一个netlink_table， ：lback函数,用于接受并处理该类型的netlink消息。 In initialization, each type netlink call netlink_kernel_create, which will prepare two parts: 1. Register in a netlink_table of nl_table, 2. Create a netlink kernel socket for the registered proto.

要组播一个netlink消息,指定groups即可。 要发一个netlink消息给特定的netlink socket只需指明其pid即可。 所有发送给内核的单播消息pid为0。 内核根据proto自动调用相应proto的callback函数， 根据具体消息内容，有的还需要kernel为处理结构回送一个新的netlinkmessage 给发送方的netlink sock。

netlink socket proto

neilink支持的proto类型非常多。且可以自定义扩展(不建议), 建议使用generic进行扩展。

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
 
以下xfrm netlink为例，进行描述。

Each proto of netlink has many groups, for example xfrm netlink:
```
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
##Data struct && method

###struct netlink_sock
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

###xfrm netlink Register

To register a new netlink proto, two elements are needed in a struct netlink_kernel_cfg, a. groups: the MAX groups in this proto. b. input: a callback funciton for all received netlink message whose type is ‘proto’. Register a netlink type with NETLINK_XFRM, which has XFRMNLGRP_MAX groups. and all the netlink message will be processed by xfrm_netlink_rcv.

We skip pernet initilization for xfrm, See XXX for detail about pernet ops. Finaly, xfrm_user_net_init is called.

### xx?
  1. xfrm_user_net_init 

  1.1 call netlink_kernel_create 

  1.2 init the xfrm netlink part of corresponding net namespace.

  2. netlink_kernel_create(__netlink_kernel_create) 

    2.1 create a netlink xfrm socket(kernel), which will be used a netlink kernel socket to rcv(queue) all the message send to “kernel/xfrm” 

    2.2 regiser the xfrm proto in the netlink_table.
`__netlink_create`[socket_sock_png]: http://martinbj2008.github.com/pic/socket_sock.png [Source]https://gist.github.com/martinbj2008/5096075#file-gistfile3-c

##user socket creation

netlink_sendmsg

netlink_sendmsg —–> netlink_unicastg —–> nlk->netlink_rcv,

Create a skb and init NETLINK_CB(skb), Copy the message data with memcpy_fromiovec.

If its destination is a group, then broadcast it with netlink_broadcast, or it is a unicast message, send it by netlink_unicast.

nlk->netlink_rcv is set in netlink_create fox xfrm netlink message, it is xfrm_netlink_rcv

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
891 int netlink_unicast(struct sock ssk, struct sk_buff skb, 892 u32 pid, int nonblock) 893 { 894 struct sock *sk; 895 int err; 896 long timeo; 897 898 skb = netlink_trim(skb, gfp_any()); 899 900 timeo = sock_sndtimeo(ssk, nonblock); 901 retry: 902 sk = netlink_getsockbypid(ssk, pid); 903 if (IS_ERR(sk)) { 904 kfree_skb(skb); 905 return PTR_ERR(sk); 906 } 907 if (netlink_is_kernel(sk)) 908 return netlink_unicast_kernel(sk, skb); 909 910 if (sk_filter(sk, skb)) { 911 err = skb->len; 912 kfree_skb(skb); 913 sock_put(sk); 914 return err; 915 } 916 917 err = netlink_attachskb(sk, skb, &timeo, ssk); 918 if (err == 1) 919 goto retry; 920 if (err) 921 return err; 922 923 return netlink_sendskb(sk, skb); 924 } 925 EXPORT_SYMBOL(netlink_unicast);

broadcast message

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

```c
101 static struct rtnl_link *rtnl_msg_handlers[RTNL_FAMILY_MAX + 1];
pernet ops
```

```c
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

```c
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

```c
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

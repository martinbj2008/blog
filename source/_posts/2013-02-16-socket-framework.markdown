---
layout: post
title: "Socket Basic Framework"
date: 2013-02-16 00:00
comments: true
categories: [socket]
tags: [kernel, socket]
---


##socket type

socket is a widely used in kernel/net. There are many kind of sockets with different ‘family/proto’. They can be divided by family, and under each family, there are many kinds of proto. such as inet(ipv4)/stream, netlink/(xfrm, rt).

<!-- more -->

##Data struct && method

###net_proto_family

The net_proto_family is used to register a socket family.

```c
216 struct net_proto_family {
217         int             family;
218         int             (*create)(struct net *net, struct socket *sock,
219                                   int protocol, int kern);
220         struct module   *owner;
221 };
```
such as inet(ipv4), netlink, unix_socket, pf_packet(tcpdump socket) … 
EX: netlink family

```c
2082 static const struct net_proto_family netlink_family_ops = {
2083         .family = PF_NETLINK,
2084         .create = netlink_create,
2085         .owner  = THIS_MODULE,  /* for consistency 8) */
2086 };
```

All the socket families are registered to net_families by int sock_register(const struct net_proto_family *ops)

```c
158 static DEFINE_SPINLOCK(net_family_lock);
159 static const struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;
```

See later section for detail.

###`struct proto_ops`

proto_ops is used to define the detail operation, such as bind/connect/accept and so on. every socket has a specificed proto_ops.
```c
162 struct proto_ops {
163         int             family;
164         struct module   *owner;
165         int             (*release)   (struct socket *sock);
166         int             (*bind)      (struct socket *sock,
167                                       struct sockaddr *myaddr,
168                                       int sockaddr_len);
169         int             (*connect)   (struct socket *sock,
170                                       struct sockaddr *vaddr,
171                                       int sockaddr_len, int flags);
172         int             (*socketpair)(struct socket *sock1,
173                                       struct socket *sock2);
174         int             (*accept)    (struct socket *sock,
175                                       struct socket *newsock, int flags);
176         int             (*getname)   (struct socket *sock,
177                                       struct sockaddr *addr,
...
199         int             (*sendmsg)   (struct kiocb *iocb, struct socket *sock,
200                                       struct msghdr *m, size_t total_len);
201         int             (*recvmsg)   (struct kiocb *iocb, struct socket *sock,
202                                       struct msghdr *m, size_t total_len,
203                                       int flags);
...
```
###struct proto

```c
 800 struct proto {
 ...
 812         int                     (*init)(struct sock *sk);
 813         void                    (*destroy)(struct sock *sk);
 814         void                    (*shutdown)(struct sock *sk, int how);
 ..
 847         /* Keeping track of sk's, looking them up, and port selection methods. */
 848         void                    (*hash)(struct sock *sk);
 849         void                    (*unhash)(struct sock *sk);
 ...
 876         struct kmem_cache       *slab;
 877         unsigned int            obj_size;
 878         int                     slab_flags;
...
 911 };
```

Currently, I only found the obj_size/slab is useful. 
```c
876 struct kmem_cache *slab; 
877 unsigned int obj_size;
```

When create a socket in socket_create, we alloc a struct sock, in fact, we alloc a memory whose size is obj_size from the slab.

ex:netlink_sock, its first element is struct sock sk;, thus we can use nlk_sk to get a struct netlink_sock from a struct sock.
```c
67 struct netlink_sock {
68         /* struct sock has to be the first member of netlink_sock */
69         struct sock             sk;
```

```c
96 static inline struct netlink_sock *nlk_sk(struct sock *sk)
97 {
98         return container_of(sk, struct netlink_sock, sk);
99 }
```

sometimes struct proto_ops will call the struct proto 
ex: .getsockopt (todo)
###socket vs sock

What is sock used for? sock is the internel representation “network layer representation of sockets” Which is used in kernel internel.

What is socket used for? “struct socket – general BSD socket” It is a very high layer abtractation, has very little and common information, such as: state, file(socket is a file for user space), flag, ops(proto ops). each of them has a pointer in its struct which points to peer.

PIC for sock and socket

###`socket->sk`
struct socket - general BSD socket
```c
139 struct socket {
140         socket_state            state;
141
142         kmemcheck_bitfield_begin(type);
143         short                   type;
144         kmemcheck_bitfield_end(type);
145
146         unsigned long           flags;
147
148         struct socket_wq __rcu  *wq;
149
150         struct file             *file;
151         struct sock             *sk;
152         const struct proto_ops  *ops;
153 };
```c

### `sock->sk_socket`

```c
264 struct sock {
...
 269         struct sock_common      __sk_common;
...
 285         socket_lock_t           sk_lock;
 286         struct sk_buff_head     sk_receive_queue;
...
 295         struct {
 296                 atomic_t        rmem_alloc;
 297                 int             len;
 298                 struct sk_buff  *head;
 299                 struct sk_buff  *tail;
 300         } sk_backlog;
...
 360         struct socket           *sk_socket;
 380 };
```

### `void sock_init_data(struct socket *sock, struct sock *sk)`

```c
2077 void sock_init_data(struct socket *sock, struct sock *sk)
2078 {
2079         skb_queue_head_init(&sk->sk_receive_queue);
2080         skb_queue_head_init(&sk->sk_write_queue);
2081         skb_queue_head_init(&sk->sk_error_queue);
2082 #ifdef CONFIG_NET_DMA
2083         skb_queue_head_init(&sk->sk_async_wait_queue);
2084 #endif
2085
2086         sk->sk_send_head        =       NULL;
2087
2088         init_timer(&sk->sk_timer);
2089
2090         sk->sk_allocation       =       GFP_KERNEL;
2091         sk->sk_rcvbuf           =       sysctl_rmem_default;
2092         sk->sk_sndbuf           =       sysctl_wmem_default;
2093         sk->sk_state            =       TCP_CLOSE;
2094         sk_set_socket(sk, sock);
2095
2096         sock_set_flag(sk, SOCK_ZAPPED);
2097
2098         if (sock) {
2099                 sk->sk_type     =       sock->type;
2100                 sk->sk_wq       =       sock->wq;
2101                 sock->sk        =       sk;
2102         } else
2103                 sk->sk_wq       =       NULL;
2104
2105         spin_lock_init(&sk->sk_dst_lock);
2106         rwlock_init(&sk->sk_callback_lock);
2107         lockdep_set_class_and_name(&sk->sk_callback_lock,
2108                         af_callback_keys + sk->sk_family,
2109                         af_family_clock_key_strings[sk->sk_family]);
2110
2111         sk->sk_state_change     =       sock_def_wakeup;
2112         sk->sk_data_ready       =       sock_def_readable;
2113         sk->sk_write_space      =       sock_def_write_space;
2114         sk->sk_error_report     =       sock_def_error_report;
2115         sk->sk_destruct         =       sock_def_destruct;
2116
2117         sk->sk_sndmsg_page      =       NULL;
2118         sk->sk_sndmsg_off       =       0;
2119         sk->sk_peek_off         =       -1;
2120
2121         sk->sk_peer_pid         =       NULL;
2122         sk->sk_peer_cred        =       NULL;
2123         sk->sk_write_pending    =       0;
2124         sk->sk_rcvlowat         =       1;
2125         sk->sk_rcvtimeo         =       MAX_SCHEDULE_TIMEOUT;
2126         sk->sk_sndtimeo         =       MAX_SCHEDULE_TIMEOUT;
2127
2128         sk->sk_stamp = ktime_set(-1L, 0);
2129
2130         /*
2131          * Before updating sk_refcnt, we must commit prior changes to memory
2132          * (Documentation/RCU/rculist_nulls.txt for details)
2133          */
2134         smp_wmb();
2135         atomic_set(&sk->sk_refcnt, 1);
2136         atomic_set(&sk->sk_drops, 0);
2137 }
2138 EXPORT_SYMBOL(sock_init_data);
```

##Socket Related Initialization

###`net_families` and `socket_register`

each of the socket family initialization, sock_register is called to register a handler.

```c
[junwei@junwei]$ grep sock_register net/ -Rwn
net/caif/caif_socket.c:1100:   int err = sock_register(&caif_family_ops);
net/econet/af_econet.c:1156:   sock_register(&econet_family_ops);
net/can/af_can.c:846:  sock_register(&can_family_ops);
net/unix/af_unix.c:2422:   sock_register(&unix_family_ops);
net/tipc/socket.c:1874:    res = sock_register(&tipc_family_ops);
net/bluetooth/af_bluetooth.c:561:  err = sock_register(&bt_sock_family_ops);
net/key/af_key.c:3797: err = sock_register(&pfkey_family_ops);
net/ipv4/af_inet.c:1668:   (void)sock_register(&inet_family_ops);
net/rose/af_rose.c:1565:   sock_register(&rose_family_ops);
net/netrom/af_netrom.c:1439:   if (sock_register(&nr_family_ops))
net/rxrpc/af_rxrpc.c:823:  ret = sock_register(&rxrpc_family_ops);
net/atm/svc.c:686: return sock_register(&svc_family_ops);
net/atm/pvc.c:156: return sock_register(&pvc_family_ops);
net/packet/af_packet.c:3961:   sock_register(&packet_family_ops);
net/decnet/af_decnet.c:2381:   sock_register(&dn_family_ops);
net/iucv/af_iucv.c:2423:   err = sock_register(&iucv_sock_family_ops);
net/x25/af_x25.c:1806: rc = sock_register(&x25_family_ops);
net/netlink/af_netlink.c:2174: sock_register(&netlink_family_ops);
net/ax25/af_ax25.c:1990:   sock_register(&ax25_family_ops);
net/nfc/af_nfc.c:93:   return sock_register(&nfc_sock_family_ops);
net/phonet/af_phonet.c:511:    err = sock_register(&phonet_proto_family);
net/socket.c:2467: *  sock_register - add a socket protocol handler
net/socket.c:2475:int sock_register(const struct net_proto_family *ops)
net/socket.c:2498:EXPORT_SYMBOL(sock_register);
net/appletalk/ddp.c:1931:  (void)sock_register(&atalk_family_ops);
net/ipx/af_ipx.c:2025: sock_register(&ipx_family_ops);
net/irda/af_irda.c:2731:       rc = sock_register(&irda_family_ops);
net/llc/af_llc.c:1217: rc = sock_register(&llc_ui_family_ops);
net/rds/af_rds.c:565:  ret = sock_register(&rds_family_ops);
net/ipv6/af_inet6.c:1110:  err = sock_register(&inet6_family_ops);
net/ieee802154/af_ieee802154.c:346:    rc = sock_register(&ieee802154_family_ops);
```

```c
2466 /**
2467  *      sock_register - add a socket protocol handler
2468  *      @ops: description of protocol
2469  *
2470  *      This function is called by a protocol handler that wants to
2471  *      advertise its address family, and have it linked into the
2472  *      socket interface. The value ops->family coresponds to the
2473  *      socket system call protocol family.
2474  */
2475 int sock_register(const struct net_proto_family *ops)
...
2485         spin_lock(&net_family_lock);
2486         if (rcu_dereference_protected(net_families[ops->family],
2487                                       lockdep_is_held(&net_family_lock)))
2488                 err = -EEXIST;
2489         else {
2490                 rcu_assign_pointer(net_families[ops->family], ops);
2491                 err = 0;
2492         }
2493         spin_unlock(&net_family_lock);
...
```
##socket related system call

###syscall
In kennel(ver 3.4), the syscall are defined in net/socket.c

```c
[junwei@junwei linux-3.4]$ grep SYSCALL_DEFINE net/socket.c -n
1325:SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
1366:SYSCALL_DEFINE4(socketpair, int, family, int, type, int, protocol,
1447:SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
1476:SYSCALL_DEFINE2(listen, int, fd, int, backlog)
1509:SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
1583:SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
1601:SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
1633:SYSCALL_DEFINE3(getsockname, int, fd, struct sockaddr __user *, usockaddr,
1664:SYSCALL_DEFINE3(getpeername, int, fd, struct sockaddr __user *, usockaddr,
1696:SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
1743:SYSCALL_DEFINE4(send, int, fd, void __user *, buff, size_t, len,
1755:SYSCALL_DEFINE6(recvfrom, int, fd, void __user *, ubuf, size_t, size,
1811:SYSCALL_DEFINE5(setsockopt, int, fd, int, level, int, optname,
1845:SYSCALL_DEFINE5(getsockopt, int, fd, int, level, int, optname,
1875:SYSCALL_DEFINE2(shutdown, int, fd, int, how)
2020:SYSCALL_DEFINE3(sendmsg, int, fd, struct msghdr __user *, msg, unsigned, flags)
2095:SYSCALL_DEFINE4(sendmmsg, int, fd, struct mmsghdr __user *, mmsg,
2195:SYSCALL_DEFINE3(recvmsg, int, fd, struct msghdr __user *, msg,
2319:SYSCALL_DEFINE5(recvmmsg, int, fd, struct mmsghdr __user *, mmsg,
2361:SYSCALL_DEFINE2(socketcall, int, call, unsigned long __user *, args)

```
ex: netlink socket register.

```c
2129 static int __init netlink_proto_init(void)
...
2174         sock_register(&netlink_family_ops);
2175         register_pernet_subsys(&netlink_net_ops);
```
###socket system call socket

`sock_create` is a very basic and important method.

When create a socket, we dedicate the its family/proto, The create method of the corresponding net_proto_family is called, and set the socket->ops with corresponding struct proto_ops.

```c
139 struct socket
...
152         const struct proto_ops  *ops;
```

See detail:

### `SYSCALL_DEFINE3(socket`
```c
1325 SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
...
1337         flags = type & ~SOCK_TYPE_MASK;
1338         if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
1339                 return -EINVAL;
1340         type &= SOCK_TYPE_MASK;
1341
1342         if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
1343                 flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;
1344
1345         retval = sock_create(family, type, protocol, &sock);
1346         if (retval < 0)
1347                 goto out;
1348
1349         retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
...
1359         return retval;
```

#### `int sock_create(...)`

```c
1313 int sock_create(int family, int type, int protocol, struct socket **res)
1314 {
1315         return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
1316 }
```
```c
1199 int __sock_create(struct net *net, int family, int type, int protocol,
...
1259         rcu_read_lock();
1260         pf = rcu_dereference(net_families[family]);
...
1272         /* Now protected by module ref count */
1273         rcu_read_unlock();
...
1275         err = pf->create(net, sock, protocol, kern);
1276         if (err < 0)
1277                 goto out_module_put;
```
NOTE:
```
1260 pf = rcu_dereference(net_families[family]); 
1275 err = pf->create(net, sock, protocol, kern);
```

What is pf->create(...)? 
For netlink socket, pf->create is netlink_create.

### socket system call listen

Most of them are simple, and just call respond function by ops->xxx. 
ex: listen

```c
1476 SYSCALL_DEFINE2(listen, int, fd, int, backlog)
...
1482         sock = sockfd_lookup_light(fd, &err, &fput_needed);
1483         if (sock) {
1484                 somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
1485                 if ((unsigned)backlog > somaxconn)
1486                         backlog = somaxconn;
1487
1488                 err = security_socket_listen(sock, backlog);
1489                 if (!err)
1490                         err = sock->ops->listen(sock, backlog);
1491
1492                 fput_light(sock->file, fput_needed);
1493         }
1494         return err;
```
The import ops is inited in function sock_create

ex: the ops in netlink socket.

```
 430 static int netlink_create(struct net *net, struct socket *sock, int protocol,
 ...
 465         err = __netlink_create(net, sock, cb_mutex, protocol);
 ...
```

```c
 402 static int __netlink_create(struct net *net, struct socket *sock,
 403                             struct mutex *cb_mutex, int protocol)
...
 408         sock->ops = &netlink_ops;
```
### other socket system call
the socket related sytem call has similar source, just like sock->ops->listen(...)

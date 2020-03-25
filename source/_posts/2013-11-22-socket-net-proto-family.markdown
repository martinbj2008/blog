---
layout: post
title: "socket net_proto_family"
date: 2013-11-22 11:29
comments: true
categories: [socket]
tags: [sock]
---

### summary
Each `family` has a corresponding array element of `struct net_proto_family`,
which will be called in system call `socket`.

### Data Structure
```c
181 struct net_proto_family {
182         int             family;
183         int             (*create)(struct net *net, struct socket *sock,
184                                   int protocol, int kern);
185         struct module   *owner;
186 };
```
The `create` is *important*, which is first and basic function during
system call `socket`.

```c
 164 static DEFINE_SPINLOCK(net_family_lock);
 165 static const struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;
```

<!-- more -->

#### NOTE:
1. A pointer array and a spin_lock to protect it.
2. The spin_lock is used for mutex of **multi writer** (register or unregister).
3. RUC lock is used for the mutex between **writer**  and **reader** .

#### Register/Unregister a net famliy's `ops`
##### sock_register
```c
2567 /**
2568  *      sock_register - add a socket protocol handler
2569  *      @ops: description of protocol
2570  *
2571  *      This function is called by a protocol handler that wants to
2572  *      advertise its address family, and have it linked into the
2573  *      socket interface. The value ops->family coresponds to the
2574  *      socket system call protocol family.
2575  */
2576 int sock_register(const struct net_proto_family *ops)
2577 {
2578         int err;
2579 
2580         if (ops->family >= NPROTO) {
2581                 printk(KERN_CRIT "protocol %d >= NPROTO(%d)\n", ops->family,
2582                        NPROTO);
2583                 return -ENOBUFS;
2584         }
2585 
2586         spin_lock(&net_family_lock);
2587         if (rcu_dereference_protected(net_families[ops->family],
2588                                       lockdep_is_held(&net_family_lock)))
2589                 err = -EEXIST;
2590         else {
2591                 rcu_assign_pointer(net_families[ops->family], ops);
2592                 err = 0;
2593         }
2594         spin_unlock(&net_family_lock);
2595 
2596         printk(KERN_INFO "NET: Registered protocol family %d\n", ops->family);
2597         return err;
2598 }
2599 EXPORT_SYMBOL(sock_register);
```

#### sock_unregister
```c
2601 /**
2602  *      sock_unregister - remove a protocol handler
2603  *      @family: protocol family to remove
2604  *
2605  *      This function is called by a protocol handler that wants to
2606  *      remove its address family, and have it unlinked from the
2607  *      new socket creation.
2608  *
2609  *      If protocol handler is a module, then it can use module reference
2610  *      counts to protect against new references. If protocol handler is not
2611  *      a module then it needs to provide its own protection in
2612  *      the ops->create routine.
2613  */
2614 void sock_unregister(int family)
2615 {
2616         BUG_ON(family < 0 || family >= NPROTO);
2617 
2618         spin_lock(&net_family_lock);
2619         RCU_INIT_POINTER(net_families[family], NULL);
2620         spin_unlock(&net_family_lock);
2621 
2622         synchronize_rcu();
2623 
2624         printk(KERN_INFO "NET: Unregistered protocol family %d\n", family);
2625 }
2626 EXPORT_SYMBOL(sock_unregister);
```

### who are registered to `net_families`
```c
net/phonet/af_phonet.c:	err = sock_register(&phonet_proto_family);
net/rds/af_rds.c:	ret = sock_register(&rds_family_ops);
net/netrom/af_netrom.c:	if (sock_register(&nr_family_ops)) {
net/ipx/af_ipx.c:	sock_register(&ipx_family_ops);
net/vmw_vsock/af_vsock.c:	err = sock_register(&vsock_family_ops);
net/iucv/af_iucv.c:	err = sock_register(&iucv_sock_family_ops);
net/caif/caif_socket.c:	int err = sock_register(&caif_family_ops);
net/decnet/af_decnet.c:	sock_register(&dn_family_ops);
net/nfc/af_nfc.c:	return sock_register(&nfc_sock_family_ops);
net/unix/af_unix.c:	sock_register(&unix_family_ops);
net/packet/af_packet.c:	sock_register(&packet_family_ops);
net/appletalk/ddp.c:	(void)sock_register(&atalk_family_ops);
net/can/af_can.c:	sock_register(&can_family_ops);
net/irda/af_irda.c:		rc = sock_register(&irda_family_ops);
net/ax25/af_ax25.c:	sock_register(&ax25_family_ops);
net/atm/svc.c:	return sock_register(&svc_family_ops);
net/atm/pvc.c:	return sock_register(&pvc_family_ops);
net/rose/af_rose.c:	sock_register(&rose_family_ops);
net/rxrpc/af_rxrpc.c:	ret = sock_register(&rxrpc_family_ops);
net/x25/af_x25.c:	rc = sock_register(&x25_family_ops);
net/netlink/af_netlink.c:	sock_register(&netlink_family_ops);
net/ipv6/af_inet6.c:	err = sock_register(&inet6_family_ops);
net/bluetooth/af_bluetooth.c:	err = sock_register(&bt_sock_family_ops);
net/tipc/socket.c:	res = sock_register(&tipc_family_ops);
net/socket.c: *	sock_register - add a socket protocol handler
net/socket.c:int sock_register(const struct net_proto_family *ops)
net/socket.c:EXPORT_SYMBOL(sock_register);
net/key/af_key.c:	err = sock_register(&pfkey_family_ops);
net/ieee802154/af_ieee802154.c:	rc = sock_register(&ieee802154_family_ops);
net/ipv4/af_inet.c:	(void)sock_register(&inet_family_ops);
net/llc/af_llc.c:	rc = sock_register(&llc_ui_family_ops);
```

For example:
    `AF_INET` and `inet_family_ops`.

```c
1019 static const struct net_proto_family inet_family_ops = {
1020         .family = PF_INET,
1021         .create = inet_create,
1022         .owner  = THIS_MODULE,
1023 };
```

register when inet init.

```c
1670 static int __init inet_init(void)
1671 {
...
1702         (void)sock_register(&inet_family_ops);
```

### `net_families` is used in system call `socket`

```c
1243 int __sock_create(struct net *net, int family, int type, int protocol,
1244                          struct socket **res, int kern)
1245 {
...
1291 #ifdef CONFIG_MODULES
1292         /* Attempt to load a protocol module if the find failed.
1293          *
1294          * 12/09/1996 Marcin: But! this makes REALLY only sense, if the user
1295          * requested real, full-featured networking support upon configuration.
1296          * Otherwise module support will break!
1297          */
1298         if (rcu_access_pointer(net_families[family]) == NULL)
1299                 request_module("net-pf-%d", family);
1300 #endif
1301 
1302         rcu_read_lock();
1303         pf = rcu_dereference(net_families[family]);
1304         err = -EAFNOSUPPORT;
1305         if (!pf)
1306                 goto out_release;
1307 
1308         /*
1309          * We will call the ->create function, that possibly is in a loadable
1310          * module, so we have to bump that loadable module refcnt first.
1311          */
1312         if (!try_module_get(pf->owner))
1313                 goto out_release;
1314 
1315         /* Now protected by module ref count */
1316         rcu_read_unlock();
```

---
layout: post
title: "system call socket"
date: 2013-11-22 11:31
comments: true
categories: [socket]
tags: [socket, system call]
---

### Summary
System call `socket` will do two things:
1. create a `struct socket *sock`.
    Mainly done by `__sock_create`, which alloc a `struct socket *sock`,
then init it with the `creating` method of `net_families[family]`.
2. map the `sock` to a file descriptor by `sock_map_fd`.
   TODO.....

### call trace:
```c
> socket
> > sock_create
> > > __sock_create
> > > > sock = sock_alloc();
> > > > sock->type = type;
> > > > rcu_read_lock();
> > > > pf = rcu_dereference(net_families[family]);
> > > > try_module_get(pf->owner))
> > > > rcu_read_unlock();
> > > > pf->create(net, sock, protocol, kern);
> > > > module_put(pf->owner)
> > socket_map_fd
```
<!-- more -->

For `AF_INET` socket `pf->create` is `inet_create`,
which will be duscussed here.

```c
1368 SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
1369 {
1370         int retval;
1371         struct socket *sock;
1372         int flags;
1373 
1374         /* Check the SOCK_* constants for consistency.  */
1375         BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
1376         BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
1377         BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
1378         BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);
1379 
1380         flags = type & ~SOCK_TYPE_MASK;
1381         if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
1382                 return -EINVAL;
1383         type &= SOCK_TYPE_MASK;
1384 
1385         if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
1386                 flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;
1387 
1388         retval = sock_create(family, type, protocol, &sock);
1389         if (retval < 0)
1390                 goto out;
1391 
1392         retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
1393         if (retval < 0)
1394                 goto out_release;
1395 
1396 out:
1397         /* It may be already another descriptor 8) Not kernel problem. */
1398         return retval;
1399 
1400 out_release:
1401         sock_release(sock);
1402         return retval;
1403 }
```
### `socket_create` and `__sock_create`
```c
1356 int sock_create(int family, int type, int protocol, struct socket **res)
1357 {
1358         return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
1359 }
1360 EXPORT_SYMBOL(sock_create);
```

```c
1243 int __sock_create(struct net *net, int family, int type, int protocol,
1244                          struct socket **res, int kern)
1245 {
1246         int err;
1247         struct socket *sock;
1248         const struct net_proto_family *pf;
1249 
1250         /*
1251          *      Check protocol is in range
1252          */
1253         if (family < 0 || family >= NPROTO)
1254                 return -EAFNOSUPPORT;
1255         if (type < 0 || type >= SOCK_MAX)
1256                 return -EINVAL;
1257 
1258         /* Compatibility.
1259 
1260            This uglymoron is moved from INET layer to here to avoid
1261            deadlock in module load.
1262          */
1263         if (family == PF_INET && type == SOCK_PACKET) {
1264                 static int warned;
1265                 if (!warned) {
1266                         warned = 1;
1267                         printk(KERN_INFO "%s uses obsolete (PF_INET,SOCK_PACKET)\n",
1268                                current->comm);
1269                 }
1270                 family = PF_PACKET;
1271         }
1272 
1273         err = security_socket_create(family, type, protocol, kern);
1274         if (err)
1275                 return err;
1276 
1277         /*
1278          *      Allocate the socket and allow the family to set things up. if
1279          *      the protocol is 0, the family is instructed to select an appropriate
1280          *      default.
1281          */
1282         sock = sock_alloc();
1283         if (!sock) {
1284                 net_warn_ratelimited("socket: no more sockets\n");
1285                 return -ENFILE; /* Not exactly a match, but its the
1286                                    closest posix thing */
1287         }
1288 
1289         sock->type = type;
1290 
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
1317 
1318         err = pf->create(net, sock, protocol, kern);
1319         if (err < 0)
1320                 goto out_module_put;
1321 
1322         /*
1323          * Now to bump the refcnt of the [loadable] module that owns this
1324          * socket at sock_release time we decrement its refcnt.
1325          */
1326         if (!try_module_get(sock->ops->owner))
1327                 goto out_module_busy;
1328 
1329         /*
1330          * Now that we're done with the ->create function, the [loadable]
1331          * module can have its refcnt decremented
1332          */
1333         module_put(pf->owner);
1334         err = security_socket_post_create(sock, family, type, protocol, kern);
1335         if (err)
1336                 goto out_sock_release;
1337         *res = sock;
1338 
1339         return 0;
1340 
1341 out_module_busy:
1342         err = -EAFNOSUPPORT;
1343 out_module_put:
1344         sock->ops = NULL;
1345         module_put(pf->owner);
1346 out_sock_release:
1347         sock_release(sock);
1348         return err;
1349 
1350 out_release:
1351         rcu_read_unlock();
1352         goto out_sock_release;
1353 }
1354 EXPORT_SYMBOL(__sock_create);
```

What is `pf->create`? for `PF_INET` family please see here.
####
```c
 528 /**
 529  *      sock_alloc      -       allocate a socket
 530  *
 531  *      Allocate a new inode and socket object. The two are bound together
 532  *      and initialised. The socket is then returned. If we are out of inodes
 533  *      NULL is returned.
 534  */
 535   
 536 static struct socket *sock_alloc(void)
 537 { 
 538         struct inode *inode;
 539         struct socket *sock;
 540   
 541         inode = new_inode_pseudo(sock_mnt->mnt_sb);
 542         if (!inode)
 543                 return NULL;
 544   
 545         sock = SOCKET_I(inode);        
 546   
 547         kmemcheck_annotate_bitfield(sock, type);
 548         inode->i_ino = get_next_ino(); 
 549         inode->i_mode = S_IFSOCK | S_IRWXUGO;
 550         inode->i_uid = current_fsuid();
 551         inode->i_gid = current_fsgid();
 552         inode->i_op = &sockfs_inode_ops;
 553   
 554         this_cpu_add(sockets_in_use, 1);
 555         return sock;     
 556 } 
```

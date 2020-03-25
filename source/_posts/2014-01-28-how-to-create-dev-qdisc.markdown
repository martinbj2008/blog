---
layout: post
title: "how to create dev qdisc"
date: 2014-01-28 13:47
comments: true
categories: [netcore]
---
## Summary

### Part 1: Register multi queue net device.
In this part, only the framework is prepared for qdisc,
and the `noop_qdisc` is set as default.

#### prepare `netdev_queue`s.
for example: intel igb hardware has 8 hardware tx queue,
and nic driver create 8 corresponding `struct netdev_queue`
in the `_tx` of `struct net_device`.

#### prepare `mq_qdisc`
The `mq_qdisc` is attached to the corresponding device.
In `mq_qdisc` private field, a default qdisc will be
create for **each** NIC's hardware queue.
This is done in `mq_init`. 
The default qdisc is `pfifo_fast_ops`.

#### attach `mq_qdisc` to `netdev_queue`.

In `mq_attach`, these qdiscs are attatched to corresponding
`struct netdev_queue`.


### Part 2: Active a net device with right qdiscs
Here only trace with the case `mq_qdisc`.
When dev is up, `dev_open` is called, which will call `dev_activate`.

<!-- more -->
#### 2.1
A qdisc with `mq_qdisc_ops` is alloced and assigned to `dev->qdisc`,
The `mq qdisc` is not a simple qdisc, which includes sub-qdisc in its 
private fields. Each sub-qdisc is a default qdisc(pfif0).
<!-- ![mq qdisc](/images/mq/mq.blankflowchar.jpeg) -->
![mq qdisc](https://www.lucidchart.com/publicSegments/view/52f496a8-5c18-4c12-9d2e-62830a0093bd/image.png)

#### 2.2
when `attach` a qdisc to a device by `qdisc->attach`(which equal `mq_attach`),
in `mq_attach`, each qdisc alloc in 2.1, will be assigned to a 
netdevice queue(which is alloced in part1).

by now, each netdevice queue has a qdisc, but the qdisc is assigned to
`qdisc_sleeping` of `netdev_queue`.

at last, `dev_activate` call `transition_one_qdisc` for each tx queue,
to set `netdev_queue->qdisc` with `netdev_queue->qdisc_sleeping`.

`attach_default_qdiscs` will be called twice time,
one is called to create with `mq_qdisc_ops`,
one is called to create with `default_qdisc_ops`,


#### Data Structure
##### `struct net_device` and `netdev_queue`
```c
 struct net_device {
...
1350         struct netdev_queue     *_tx ____cacheline_aligned_in_smp;
1351
1352         /* Number of TX queues allocated at alloc_netdev_mq() time  */
1353         unsigned int            num_tx_queues;
```

```c
 544 struct netdev_queue {
 ...
 548         struct net_device       *dev;
 549         struct Qdisc            *qdisc;
 550         struct Qdisc            *qdisc_sleeping;
...
 560         spinlock_t              _xmit_lock ____cacheline_aligned_in_smp;
...
 573         unsigned long           state;
```

```c
 21 struct mq_sched {
 22         struct Qdisc            **qdiscs;
 23 };
```

#### Call trace.
##### Part 1
```c
> igb_probe
> > alloc_etherdev_mq
> > > alloc_etherdev_mqs
> > > > alloc_netdev_mqs
> > > > > ether_setup
> > > > > netif_alloc_netdev_queues
> > > > > > netdev_for_each_tx_queue(dev, netdev_init_one_queue, NULL);
> > > > > > > netdev_init_one_queue
> > register_netdev
> > > register_netdevice
> > > > dev_init_scheduler
> > > > > netdev_for_each_tx_queue(dev, dev_init_scheduler_queue, &noop_qdisc);
> > > > > > dev_init_scheduler_queue
```

NOTE:
if disable RPS(google patch), there is no soft queue to receive packet.
because there is `igb_ring` in the driver.

###### Part 2
```c
> dev_open
> > __dev_open
> > > dev_activate
> > > > attach_default_qdiscs(dev);
> > > > > qdisc_create_dflt(txq, &mq_qdisc_ops, TC_H_ROOT);
> > > > > > qdisc_alloc
> > > > > > ops->init(equal: mq_init)
> > > > > > > for (ntx = 0; ntx < dev->num_tx_queues; ntx++) {
> > > > > > > > qdisc_create_dflt(dev_queue, default_qdisc_ops ...
> > > > > > > > > qdisc_alloc
> > > > > > > > > ops->init
> > > > > qdisc->ops->attach
> > > > > dev_graft_qdisc
> > > > netdev_for_each_tx_queue(dev, transition_one_qdisc, &need_watchdog); 
> > > > > transition_one_qdisc
```

### Functions in Part 1.
```c
2016 static int igb_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
2017 {
...
2067         netdev = alloc_etherdev_mq(sizeof(struct igb_adapter),
2068                                    IGB_MAX_TX_QUEUES);
2069         if (!netdev)
2070                 goto err_alloc_etherdev;
...
2317         strcpy(netdev->name, "eth%d");
2318         err = register_netdev(netdev);
2319         if (err)
2320                 goto err_register;
...
```

```c
 51 #define alloc_etherdev_mq(sizeof_priv, count) alloc_etherdev_mqs(sizeof_priv, count, count)
```

```c
372 /**
373  * alloc_etherdev_mqs - Allocates and sets up an Ethernet device
374  * @sizeof_priv: Size of additional driver-private structure to be allocated
375  *      for this Ethernet device
376  * @txqs: The number of TX queues this device has.
377  * @rxqs: The number of RX queues this device has.
378  *
379  * Fill in the fields of the device structure with Ethernet-generic
380  * values. Basically does everything except registering the device.
381  *
382  * Constructs a new net device, complete with a private data area of
383  * size (sizeof_priv).  A 32-byte (not bit) alignment is enforced for
384  * this private data area.
385  */
386
387 struct net_device *alloc_etherdev_mqs(int sizeof_priv, unsigned int txqs,
388                                       unsigned int rxqs)
389 {
390         return alloc_netdev_mqs(sizeof_priv, "eth%d", ether_setup, txqs, rxqs);
391 }
392 EXPORT_SYMBOL(alloc_etherdev_mqs);
```

```c
6223 /**
6224  *      alloc_netdev_mqs - allocate network device
6225  *      @sizeof_priv:   size of private data to allocate space for
6226  *      @name:          device name format string
6227  *      @setup:         callback to initialize device
6228  *      @txqs:          the number of TX subqueues to allocate
6229  *      @rxqs:          the number of RX subqueues to allocate
6230  *
6231  *      Allocates a struct net_device with private data area for driver use
6232  *      and performs basic initialization.  Also allocates subquue structs
6233  *      for each queue on the device.
6234  */
6235 struct net_device *alloc_netdev_mqs(int sizeof_priv, const char *name,
6236                 void (*setup)(struct net_device *),
6237                 unsigned int txqs, unsigned int rxqs)
6238 {
6239         struct net_device *dev;
6240         size_t alloc_size;
6241         struct net_device *p;
6242
6243         BUG_ON(strlen(name) >= sizeof(dev->name));
6244
6245         if (txqs < 1) {
6246                 pr_err("alloc_netdev: Unable to allocate device with zero queues\n");
6247                 return NULL;
6248         }
6249
6250 #ifdef CONFIG_RPS
6251         if (rxqs < 1) {
6252                 pr_err("alloc_netdev: Unable to allocate device with zero RX queues\n");
6253                 return NULL;
6254         }
6255 #endif
6256
6257         alloc_size = sizeof(struct net_device);
6258         if (sizeof_priv) {
6259                 /* ensure 32-byte alignment of private area */
6260                 alloc_size = ALIGN(alloc_size, NETDEV_ALIGN);
6261                 alloc_size += sizeof_priv;
6262         }
6263         /* ensure 32-byte alignment of whole construct */
6264         alloc_size += NETDEV_ALIGN - 1;
6265
6266         p = kzalloc(alloc_size, GFP_KERNEL | __GFP_NOWARN | __GFP_REPEAT);
6267         if (!p)
6268                 p = vzalloc(alloc_size);
6269         if (!p)
6270                 return NULL;
6271
6272         dev = PTR_ALIGN(p, NETDEV_ALIGN);
6273         dev->padded = (char *)dev - (char *)p;
6274
6275         dev->pcpu_refcnt = alloc_percpu(int);
6276         if (!dev->pcpu_refcnt)
6277                 goto free_dev;
6278
6279         if (dev_addr_init(dev))
6280                 goto free_pcpu;
6281
6282         dev_mc_init(dev);
6283         dev_uc_init(dev);
6284
6285         dev_net_set(dev, &init_net);
6286
6287         dev->gso_max_size = GSO_MAX_SIZE;
6288         dev->gso_max_segs = GSO_MAX_SEGS;
6289
6290         INIT_LIST_HEAD(&dev->napi_list);
6291         INIT_LIST_HEAD(&dev->unreg_list);
6292         INIT_LIST_HEAD(&dev->close_list);
6293         INIT_LIST_HEAD(&dev->link_watch_list);
6294         INIT_LIST_HEAD(&dev->adj_list.upper);
6295         INIT_LIST_HEAD(&dev->adj_list.lower);
6296         INIT_LIST_HEAD(&dev->all_adj_list.upper);
6297         INIT_LIST_HEAD(&dev->all_adj_list.lower);
6298         dev->priv_flags = IFF_XMIT_DST_RELEASE;
6299         setup(dev);
6300
6301         dev->num_tx_queues = txqs;
6302         dev->real_num_tx_queues = txqs;
6303         if (netif_alloc_netdev_queues(dev))
6304                 goto free_all;
6305
6306 #ifdef CONFIG_RPS
6307         dev->num_rx_queues = rxqs;
6308         dev->real_num_rx_queues = rxqs;
6309         if (netif_alloc_rx_queues(dev))
6310                 goto free_all;
6311 #endif
6312
6313         strcpy(dev->name, name);
6314         dev->group = INIT_NETDEV_GROUP;
6315         if (!dev->ethtool_ops)
6316                 dev->ethtool_ops = &default_ethtool_ops;
6317         return dev;
6318
6319 free_all:
6320         free_netdev(dev);
6321         return NULL;
6322
6323 free_pcpu:
6324         free_percpu(dev->pcpu_refcnt);
6325         netif_free_tx_queues(dev);
6326 #ifdef CONFIG_RPS
6327         kfree(dev->_rx);
6328 #endif
6329
6330 free_dev:
6331         netdev_freemem(dev);
6332         return NULL;
6333 }
6334 EXPORT_SYMBOL(alloc_netdev_mqs);
```

```c
5742 static int netif_alloc_netdev_queues(struct net_device *dev)
5743 {
5744         unsigned int count = dev->num_tx_queues;
5745         struct netdev_queue *tx;
5746         size_t sz = count * sizeof(*tx);
5747
5748         BUG_ON(count < 1 || count > 0xffff);
5749
5750         tx = kzalloc(sz, GFP_KERNEL | __GFP_NOWARN | __GFP_REPEAT);
5751         if (!tx) {
5752                 tx = vzalloc(sz);
5753                 if (!tx)
5754                         return -ENOMEM;
5755         }
5756         dev->_tx = tx;
5757
5758         netdev_for_each_tx_queue(dev, netdev_init_one_queue, NULL);
5759         spin_lock_init(&dev->tx_global_lock);
5760
5761         return 0;
5762 }
```

```c
5720 static void netdev_init_one_queue(struct net_device *dev,
5721                                   struct netdev_queue *queue, void *_unused)
5722 {
5723         /* Initialize queue lock */
5724         spin_lock_init(&queue->_xmit_lock);
5725         netdev_set_xmit_lockdep_class(&queue->_xmit_lock, dev->type);
5726         queue->xmit_lock_owner = -1;
5727         netdev_queue_numa_node_write(queue, NUMA_NO_NODE);
5728         queue->dev = dev;
5729 #ifdef CONFIG_BQL
5730         dql_init(&queue->dql, HZ);
5731 #endif
5732 }
```

```c
5972 int register_netdev(struct net_device *dev)
5973 {
5974         int err;
5975
5976         rtnl_lock();
5977         err = register_netdevice(dev);
5978         rtnl_unlock();
5979         return err;
5980 }
5981 EXPORT_SYMBOL(register_netdev);
```

```c
5781 int register_netdevice(struct net_device *dev)
5782 {
...
5879         linkwatch_init_dev(dev);
5880
5881         dev_init_scheduler(dev);
5882         dev_hold(dev);
5883         list_netdevice(dev);
5884         add_device_randomness(dev->dev_addr, dev->addr_len);
...
```

```c
876 void dev_init_scheduler(struct net_device *dev)
877 {
878         dev->qdisc = &noop_qdisc;
879         netdev_for_each_tx_queue(dev, dev_init_scheduler_queue, &noop_qdisc);
880         if (dev_ingress_queue(dev))
881                 dev_init_scheduler_queue(dev, dev_ingress_queue(dev), &noop_qdisc);
882
883         setup_timer(&dev->watchdog_timer, dev_watchdog, (unsigned long)dev);
884 }
```

```
866 static void dev_init_scheduler_queue(struct net_device *dev,
867                                      struct netdev_queue *dev_queue,
868                                      void *_qdisc)
869 {
870         struct Qdisc *qdisc = _qdisc;
871
872         dev_queue->qdisc = qdisc;
873         dev_queue->qdisc_sleeping = qdisc;
874 }
```

```c
366 struct Qdisc noop_qdisc = {
367         .enqueue        =       noop_enqueue,
368         .dequeue        =       noop_dequeue,
369         .flags          =       TCQ_F_BUILTIN,
370         .ops            =       &noop_qdisc_ops,
371         .list           =       LIST_HEAD_INIT(noop_qdisc.list),
372         .q.lock         =       __SPIN_LOCK_UNLOCKED(noop_qdisc.q.lock),
373         .dev_queue      =       &noop_netdev_queue,
374         .busylock       =       __SPIN_LOCK_UNLOCKED(noop_qdisc.busylock),
375 };
```
#### functions for Part 2.

```c
745 void dev_activate(struct net_device *dev)
746 {
747         int need_watchdog;
748
749         /* No queueing discipline is attached to device;
750          * create default one for devices, which need queueing
751          * and noqueue_qdisc for virtual interfaces
752          */
753
754         if (dev->qdisc == &noop_qdisc)
755                 attach_default_qdiscs(dev);
756
757         if (!netif_carrier_ok(dev))
758                 /* Delay activation until next carrier-on event */
759                 return;
760
761         need_watchdog = 0;
762         netdev_for_each_tx_queue(dev, transition_one_qdisc, &need_watchdog);
763         if (dev_ingress_queue(dev))
764                 transition_one_qdisc(dev, dev_ingress_queue(dev), NULL);
765
766         if (need_watchdog) {
767                 dev->trans_start = jiffies;
768                 dev_watchdog_up(dev);
769         }
770 }
771 EXPORT_SYMBOL(dev_activate);
```

```c
708 static void attach_default_qdiscs(struct net_device *dev)
709 {
710         struct netdev_queue *txq;
711         struct Qdisc *qdisc;
712
713         txq = netdev_get_tx_queue(dev, 0);
714
715         if (!netif_is_multiqueue(dev) || dev->tx_queue_len == 0) {
716                 netdev_for_each_tx_queue(dev, attach_one_default_qdisc, NULL);
717                 dev->qdisc = txq->qdisc_sleeping;
718                 atomic_inc(&dev->qdisc->refcnt);
719         } else {
720                 qdisc = qdisc_create_dflt(txq, &mq_qdisc_ops, TC_H_ROOT);
721                 if (qdisc) {
722                         qdisc->ops->attach(qdisc);
723                         dev->qdisc = qdisc;
724                 }
725         }
726 }
```

```c
223 static const struct Qdisc_class_ops mq_class_ops = {
224         .select_queue   = mq_select_queue,
225         .graft          = mq_graft,
226         .leaf           = mq_leaf,
227         .get            = mq_get,
228         .put            = mq_put,
229         .walk           = mq_walk,
230         .dump           = mq_dump_class,
231         .dump_stats     = mq_dump_class_stats,
232 };
233
234 struct Qdisc_ops mq_qdisc_ops __read_mostly = {
235         .cl_ops         = &mq_class_ops,
236         .id             = "mq",
237         .priv_size      = sizeof(struct mq_sched),
238         .init           = mq_init,
239         .destroy        = mq_destroy,
240         .attach         = mq_attach,
241         .dump           = mq_dump,
242         .owner          = THIS_MODULE,
243 };
```

```c
584 struct Qdisc *qdisc_create_dflt(struct netdev_queue *dev_queue,
585                                 const struct Qdisc_ops *ops,
586                                 unsigned int parentid)
587 {
588         struct Qdisc *sch;
589
590         if (!try_module_get(ops->owner))
591                 goto errout;
592
593         sch = qdisc_alloc(dev_queue, ops);
594         if (IS_ERR(sch))
595                 goto errout;
596         sch->parent = parentid;
597
598         if (!ops->init || ops->init(sch, NULL) == 0)
599                 return sch;
600
601         qdisc_destroy(sch);
602 errout:
603         return NULL;
604 }
```

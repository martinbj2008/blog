---
layout: post
title: "unregister a net device"
date: 2013-08-02 14:08
comments: true
categories: [netcore]
tags: [kernel, net, netdev]
---

`unregister_netdev` is used to delete a net device. In fact, it equals:
```c
	rtnl_lock();
	rollback_registered_many( a temp list with a single net device);
	list_add_tail(&dev->todo_list, &net_todo_list);
	rtnl_unlock();
```
a temporary list stores a single net device, which is to be deleted.
`net_todo_list` stores all the net devices are **being** deleted.

The core function is `rollback_registered_many`, which efficiently deletes many devices in a list.
But here, in this case, one a single netdevice in the list.

<!-- more -->

1. firstly check the status and confirm the net device is registered(NETREG_REGISTERED),
or it will do nothing for the net device.

2. for each net device, close it by `dev_close_many`, and notify(broadcast) 'NETDEV_DOWN' message by rtmsg and 
net device notify.

3. remove netdevice from kernel net device list (by name, ifindex, ..).

4. net device bemomes NETREG_UNREGISTERING.

5. synchronize_net();

6. `dev_shutdown` shut down net device tx/rx queue.

7.  call `call_netdevice_notifiers` with  NETDEV_UNREGISTER

8. Flush the unicast and multicast chains, remove sys, ..

9. dev_put (here only decrease reference);

in the finally, do rtnl_unlock, which do more than unlock rtnl.

it will check all the devices in the net_todo_list and wait them to be freed.
to enhace efficience, net_todo_list will be copied(moved) to a temp list,
and empty net_todo_list. thus rtnl lock is freed.

then for each netdevice in the temp list, wait them to be free(un-reference),
and then free them(deconstruct).
for safe, we Rebroadcast unregister notification every 250ms(try within 10s), during wait dev reference.

##calltrace
```c
> unregister_netdev
> > rtnl_lock();
> > unregister_netdevice(dev);
> > > unregister_netdevice_queue
> > > > rollback_registered
> > > > > list_add(&dev->unreg_list, &single);
> > > > > rollback_registered_many(&single);
> > > > > > confirm each dev->reg_state is NETREG_REGISTERED
> > > > > > dev_close_many(head);
> > > > > > list_for_each_entry(dev, head, unreg_list)
> > > > > > > unlist_netdevice(dev);
> > > > > > > dev->reg_state = NETREG_UNREGISTERING;
> > > > > > synchronize_net();
> > > > > > list_for_each_entry(dev, head, unreg_list)
> > > > > > > dev_shutdown(dev);
> > > > > > > call_netdevice_notifiers(NETDEV_UNREGISTER, dev);
> > > > > > > rtmsg_ifinfo(RTM_DELLINK, dev, ~0U); ???
> > > > > > > dev_uc_flush(dev);
> > > > > > > dev_mc_flush(dev);
> > > > > > > dev->netdev_ops->ndo_uninit(dev);
> > > > > > > netdev_unregister_kobject(dev); 
> > > > > > > netif_reset_xps_queues_gt(dev, 0);
> > > > > > synchronize_net();
> > > > > > list_for_each_entry(dev, head, unreg_list)
> > > > > > > dev_put(dev);
> > > > net_set_todo
> > > > list_add_tail(&dev->todo_list, &net_todo_list);
> > rtnl_unlock();
> > > list_replace_init(&net_todo_list, &list);
> > > __rtnl_unlock();
> > > ... real net dev destruct/free...
}
```


```c
5976 /**
5977  *      unregister_netdev - remove device from the kernel
5978  *      @dev: device
5979  *
5980  *      This function shuts down a device interface and removes it
5981  *      from the kernel tables.
5982  *
5983  *      This is just a wrapper for unregister_netdevice that takes
5984  *      the rtnl semaphore.  In general you want to use this and not
5985  *      unregister_netdevice.  
5986  */
5987 void unregister_netdev(struct net_device *dev)
5988 { 
5989         rtnl_lock();
5990         unregister_netdevice(dev);
5991         rtnl_unlock();
5992 } 
5993 EXPORT_SYMBOL(unregister_netdev);
```

```c
1749 static inline void unregister_netdevice(struct net_device *dev)
1750 {              
1751         unregister_netdevice_queue(dev, NULL);
1752 }  
```

head is NUll for unregister_netdevice case.

```c
5946 void unregister_netdevice_queue(struct net_device *dev, struct list_head *head)
5947 {
5948         ASSERT_RTNL();
5949 
5950         if (head) {
5951                 list_move_tail(&dev->unreg_list, head);
5952         } else {
5953                 rollback_registered(dev);
5954                 /* Finish processing unregister after unlock */ 
5955                 net_set_todo(dev);
5956         }
5957 }
5958 EXPORT_SYMBOL(unregister_netdevice_queue);
```

```c
5094 static void rollback_registered(struct net_device *dev)
5095 {
5096         LIST_HEAD(single);
5097 
5098         list_add(&dev->unreg_list, &single);
5099         rollback_registered_many(&single);
5100         list_del(&single);
5101 }
```

```c
5018 static void rollback_registered_many(struct list_head *head)
5019 {
5020         struct net_device *dev, *tmp;
5021 
5022         BUG_ON(dev_boot_phase);
5023         ASSERT_RTNL();
5024 
5025         list_for_each_entry_safe(dev, tmp, head, unreg_list) {
5026                 /* Some devices call without registering
5027                  * for initialization unwind. Remove those
5028                  * devices and proceed with the remaining.
5029                  */
5030                 if (dev->reg_state == NETREG_UNINITIALIZED) {
5031                         pr_debug("unregister_netdevice: device %s/%p never was registered\n",
5032                                  dev->name, dev);
5033 
5034                         WARN_ON(1);
5035                         list_del(&dev->unreg_list);
5036                         continue;
5037                 }
5038                 dev->dismantle = true;
5039                 BUG_ON(dev->reg_state != NETREG_REGISTERED);
5040         }
5041 
5042         /* If device is running, close it first. */
5043         dev_close_many(head);
5044 
5045         list_for_each_entry(dev, head, unreg_list) {
5046                 /* And unlink it from device chain. */
5047                 unlist_netdevice(dev);
5048 
5049                 dev->reg_state = NETREG_UNREGISTERING;
5050         }
5051 
5052         synchronize_net();
5053 
5054         list_for_each_entry(dev, head, unreg_list) {
5055                 /* Shutdown queueing discipline. */
5056                 dev_shutdown(dev);
5057 
5058 
5059                 /* Notify protocols, that we are about to destroy
5060                    this device. They should clean all the things.
5061                 */
5062                 call_netdevice_notifiers(NETDEV_UNREGISTER, dev);
5063 
5064                 if (!dev->rtnl_link_ops ||
5065                     dev->rtnl_link_state == RTNL_LINK_INITIALIZED)
5065                     dev->rtnl_link_state == RTNL_LINK_INITIALIZED)
5066                         rtmsg_ifinfo(RTM_DELLINK, dev, ~0U);
5067 
5068                 /*
5069                  *      Flush the unicast and multicast chains
5070                  */
5071                 dev_uc_flush(dev);
5072                 dev_mc_flush(dev);
5073 
5074                 if (dev->netdev_ops->ndo_uninit)
5075                         dev->netdev_ops->ndo_uninit(dev);
5076 
5077                 /* Notifier chain MUST detach us all upper devices. */
5078                 WARN_ON(netdev_has_any_upper_dev(dev));
5079 
5080                 /* Remove entries from kobject tree */
5081                 netdev_unregister_kobject(dev);
5082 #ifdef CONFIG_XPS
5083                 /* Remove XPS queueing entries */
5084                 netif_reset_xps_queues_gt(dev, 0);
5085 #endif
5086         }
5087 
5088         synchronize_net();
5089 
5090         list_for_each_entry(dev, head, unreg_list)
5091                 dev_put(dev);
5092 }
```

```c
1362 static int dev_close_many(struct list_head *head)
1363 {
1364         struct net_device *dev, *tmp;
1365         LIST_HEAD(tmp_list);
1366 
1367         list_for_each_entry_safe(dev, tmp, head, unreg_list)
1368                 if (!(dev->flags & IFF_UP))
1369                         list_move(&dev->unreg_list, &tmp_list);
1370 
1371         __dev_close_many(head);
1372 
1373         list_for_each_entry(dev, head, unreg_list) {
1374                 rtmsg_ifinfo(RTM_NEWLINK, dev, IFF_UP|IFF_RUNNING);
1375                 call_netdevice_notifiers(NETDEV_DOWN, dev);
1376         }
1377 
1378         /* rollback_registered_many needs the complete original list */
1379         list_splice(&tmp_list, head);
1380         return 0;
1381 }
```

```c
1303 static int __dev_close_many(struct list_head *head)
1304 {
1305         struct net_device *dev;
1306 
1307         ASSERT_RTNL();
1308         might_sleep();
1309 
1310         list_for_each_entry(dev, head, unreg_list) {
1311                 call_netdevice_notifiers(NETDEV_GOING_DOWN, dev);
1312 
1313                 clear_bit(__LINK_STATE_START, &dev->state);
1314 
1315                 /* Synchronize to scheduled poll. We cannot touch poll list, it
1316                  * can be even on different cpu. So just clear netif_running().
1317                  *
1318                  * dev->stop() will invoke napi_disable() on all of it's
1319                  * napi_struct instances on this device.
1320                  */
1321                 smp_mb__after_clear_bit(); /* Commit netif_running(). */
1322         }
1323 
1324         dev_deactivate_many(head);
1325 
1326         list_for_each_entry(dev, head, unreg_list) {
1327                 const struct net_device_ops *ops = dev->netdev_ops;
1328 
1329                 /*
1330                  *      Call the device specific close. This cannot fail.
1331                  *      Only if device is UP
1332                  *
1333                  *      We allow it to be called even after a DETACH hot-plug
1334                  *      event.
1335                  */
1336                 if (ops->ndo_stop)
1337                         ops->ndo_stop(dev);
1338 
1339                 dev->flags &= ~IFF_UP;
1340                 net_dmaengine_put();
1341         }
1342 
1343         return 0;
1344 }
```

### todo what is `dev->dismantl` 

```c
809 /**
810  *      dev_deactivate_many - deactivate transmissions on several devices
811  *      @head: list of devices to deactivate
812  *
813  *      This function returns only when all outstanding transmissions
814  *      have completed, unless all devices are in dismantle phase.
815  */
816 void dev_deactivate_many(struct list_head *head)
817 {
818         struct net_device *dev;
819         bool sync_needed = false;
820 
821         list_for_each_entry(dev, head, unreg_list) {
822                 netdev_for_each_tx_queue(dev, dev_deactivate_queue,
823                                          &noop_qdisc);
824                 if (dev_ingress_queue(dev))
825                         dev_deactivate_queue(dev, dev_ingress_queue(dev),
826                                              &noop_qdisc);
827 
828                 dev_watchdog_down(dev);
829                 sync_needed |= !dev->dismantle;
830         }
831 
832         /* Wait for outstanding qdisc-less dev_queue_xmit calls.
833          * This is avoided if all devices are in dismantle phase :
834          * Caller will call synchronize_net() for us
835          */
836         if (sync_needed)
837                 synchronize_net();
838 
839         /* Wait for outstanding qdisc_run calls. */
840         list_for_each_entry(dev, head, unreg_list)
841                 while (some_qdisc_is_busy(dev))
842                         yield();
843 }
```

```c
761 static void dev_deactivate_queue(struct net_device *dev,
762                                  struct netdev_queue *dev_queue,
763                                  void *_qdisc_default)
764 {
765         struct Qdisc *qdisc_default = _qdisc_default;
766         struct Qdisc *qdisc;
767 
768         qdisc = dev_queue->qdisc;
769         if (qdisc) {
770                 spin_lock_bh(qdisc_lock(qdisc));
771 
772                 if (!(qdisc->flags & TCQ_F_BUILTIN))
773                         set_bit(__QDISC_STATE_DEACTIVATED, &qdisc->state);
774 
775                 rcu_assign_pointer(dev_queue->qdisc, qdisc_default);
776                 qdisc_reset(qdisc);
777 
778                 spin_unlock_bh(qdisc_lock(qdisc));
779         }
780 }
```

```c
875 static void shutdown_scheduler_queue(struct net_device *dev,
876                                      struct netdev_queue *dev_queue,
877                                      void *_qdisc_default)
878 {
879         struct Qdisc *qdisc = dev_queue->qdisc_sleeping;
880         struct Qdisc *qdisc_default = _qdisc_default;
881 
882         if (qdisc) {
883                 rcu_assign_pointer(dev_queue->qdisc, qdisc_default);
884                 dev_queue->qdisc_sleeping = qdisc_default;
885 
886                 qdisc_destroy(qdisc);
887         }
888 }
```

```c
5010 /* Delayed registration/unregisteration */
5011 static LIST_HEAD(net_todo_list);
5012   
5013 static void net_set_todo(struct net_device *dev)
5014 { 
5015         list_add_tail(&dev->todo_list, &net_todo_list);
5016 }
```

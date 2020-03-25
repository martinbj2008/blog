---
layout: post
title: "netdevice watchdog cause tx queue schedule"
date: 2014-09-17 15:38
comments: true
categories: [netcore]
tags: [softirq, watchdog]
---

### test case
For ixgbe nic, we want to assign a tx hardware qeueue to each cpu,
and the tx softirq should use the corresponding hardware queue.

each packet will select a softqueue in `dev_queue_xmit`,
we rewrite ixgbe driver `ndo_select_queue`(`ixgbe_select_queue`),
which will return current cpu index(based 0) when packet select queue.
thus for each cpu use its own tx queue.

but, we found some packet had unmatched queue index when send
on specific cpu.

for example, a packet's queue index is 5 but is sent by cpu3,
thus, cpu3 will operate tx hw queue5, which should only be done by cpu5.

<!-- more -->

### Analysis
When watchdog is start, it first `freeze` all subqueues,
and the do the check.
At the end, it resume the subqueues,
and reschedule them.

Because the watchdog is handled in a timer,
so the reschedule the queue will be done on a different cpu,
which is different the packets's queue index.

for example:
packet rung select queue on CPU1, while CPU2 run the watchdog,
this packet will be store in the queue1, but not sent.
when cpu2 finish the watchdog, queue1 is rescheduled.
NOTE here the queue1 start run on cpu2 not cpu1.
which is not expected and safe.
it will cause the tx ring buffer hang.


### related source
```
232 static void dev_watchdog(unsigned long arg)
233 {
234         struct net_device *dev = (struct net_device *)arg;
235
236         netif_tx_lock(dev);
237         if (!qdisc_tx_is_noop(dev)) {
238                 if (netif_device_present(dev) &&
239                     netif_running(dev) &&
240                     netif_carrier_ok(dev)) {
241                         int some_queue_timedout = 0;
242                         unsigned int i;
243                         unsigned long trans_start;
244
245                         for (i = 0; i < dev->num_tx_queues; i++) {
246                                 struct netdev_queue *txq;
247
248                                 txq = netdev_get_tx_queue(dev, i);
249                                 /*
250                                  * old device drivers set dev->trans_start
251                                  */
252                                 trans_start = txq->trans_start ? : dev->trans_start;
253                                 if (netif_xmit_stopped(txq) &&
254                                     time_after(jiffies, (trans_start +
255                                                          dev->watchdog_timeo))) {
256                                         some_queue_timedout = 1;
257                                         txq->trans_timeout++;
258                                         break;
259                                 }
260                         }
261
262                         if (some_queue_timedout) {
263                                 WARN_ONCE(1, KERN_INFO "NETDEV WATCHDOG: %s (%s): transmit queue %u timed out\n",
264                                        dev->name, netdev_drivername(dev), i);
265                                 dev->netdev_ops->ndo_tx_timeout(dev);
266                         }
267                         if (!mod_timer(&dev->watchdog_timer,
268                                        round_jiffies(jiffies +
269                                                      dev->watchdog_timeo)))
270                                 dev_hold(dev);
271                 }
272         }
273         netif_tx_unlock(dev);
274
275         dev_put(dev);
276 }
```

```
2985 static inline void netif_tx_lock(struct net_device *dev)
2986 {
2987         unsigned int i;
2988         int cpu;
2989
2990         spin_lock(&dev->tx_global_lock);
2991         cpu = smp_processor_id();
2992         for (i = 0; i < dev->num_tx_queues; i++) {
2993                 struct netdev_queue *txq = netdev_get_tx_queue(dev, i);
2994
2995                 /* We are the only thread of execution doing a
2996                  * freeze, but we have to grab the _xmit_lock in
2997                  * order to synchronize with threads which are in
2998                  * the ->hard_start_xmit() handler and already
2999                  * checked the frozen bit.
3000                  */
3001                 __netif_tx_lock(txq, cpu);
3002                 set_bit(__QUEUE_STATE_FROZEN, &txq->state);
3003                 __netif_tx_unlock(txq);
3004         }
3005 }
```

```
3013 static inline void netif_tx_unlock(struct net_device *dev)
3014 {
3015         unsigned int i;
3016
3017         for (i = 0; i < dev->num_tx_queues; i++) {
3018                 struct netdev_queue *txq = netdev_get_tx_queue(dev, i);
3019
3020                 /* No need to grab the _xmit_lock here.  If the
3021                  * queue is not stopped for another reason, we
3022                  * force a schedule.
3023                  */
3024                 clear_bit(__QUEUE_STATE_FROZEN, &txq->state);
3025                 netif_schedule_queue(txq);
3026         }
3027         spin_unlock(&dev->tx_global_lock);
3028 }
```

```
2265 static inline void netif_schedule_queue(struct netdev_queue *txq)
2266 {
2267         if (!(txq->state & QUEUE_STATE_ANY_XOFF))
2268                 __netif_schedule(txq->qdisc);
2269 }
```

```
2150 static inline void __netif_reschedule(struct Qdisc *q)
2151 {
2152         struct softnet_data *sd;
2153         unsigned long flags;
2154
2155         local_irq_save(flags);
2156         sd = &__get_cpu_var(softnet_data);
2157         q->next_sched = NULL;
2158         *sd->output_queue_tailp = q;
2159         sd->output_queue_tailp = &q->next_sched;
2160         raise_softirq_irqoff(NET_TX_SOFTIRQ);
2161         local_irq_restore(flags);
2162 }
2163
2164 void __netif_schedule(struct Qdisc *q)
2165 {
2166         if (!test_and_set_bit(__QDISC_STATE_SCHED, &q->state))
2167                 __netif_reschedule(q);
2168 }
2169 EXPORT_SYMBOL(__netif_schedule);
```

---
layout: post
title: "how to xmit a packet with Qdisc"
date: 2014-02-08 11:26
comments: true
categories: [netcore]
tags:	[net, qdisc]
---

### summary
We think it as a ideal and simple case:


### Call Trace:
```c
> dev_queue_xmit
> >  __dev_queue_xmit(skb, NULL);
> > > rcu_read_lock_bh();
> > > txq = netdev_pick_tx(dev, skb, accel_priv);
> > > q = rcu_dereference_bh(txq->qdisc);
> > > rc = __dev_xmit_skb(skb, q, dev, txq);
> > > > skb_dst_force(skb);
> > > > q->enqueue(skb, q);
> > > > qdisc_run_begin(q)
> > > >  __qdisc_run(q);
> > > > > while (qdisc_restart(q))
> > > > > > __netif_schedule
> > > > > qdisc_run_end(q)
> > > rcu_read_unlock_bh();
> > > return rc;
```
<!-- more -->

### functions

#### `__dev_queue_xmit`

```c
2806 int __dev_queue_xmit(struct sk_buff *skb, void *accel_priv)
2807 {
2808         struct net_device *dev = skb->dev;
2809         struct netdev_queue *txq;
2810         struct Qdisc *q;
2811         int rc = -ENOMEM;
...
2818         rcu_read_lock_bh();
2819
2820         skb_update_prio(skb);
2821
2822         txq = netdev_pick_tx(dev, skb, accel_priv);
2823         q = rcu_dereference_bh(txq->qdisc);
2824
2825 #ifdef CONFIG_NET_CLS_ACT
2826         skb->tc_verd = SET_TC_AT(skb->tc_verd, AT_EGRESS);
2827 #endif
2828         trace_net_dev_queue(skb);
2829         if (q->enqueue) {
2830                 rc = __dev_xmit_skb(skb, q, dev, txq);
2831                 goto out;
2832         }
...
2883 out:
2884         rcu_read_unlock_bh();
2885         return rc;
2886 }
```

### How to schedlule Qdisc

#### case 1: empty qdisc and qdisc could be bypass
If the qdisc could be bypass, such as fifo qdisc,
and it is a empty qdisc,
and the qdisc is not running,

set the qdisc as running,
then send the packet directly by `sch_direct_xmit`.
If send success, clear the running flag by `qdisc_run_end`,
or(send failed), put the skb to qdisc queue by `dev_requeue_skb`.

#### case 2: enqueue and then send
In this case, skb must firstly enqueue.
Check and confirm qdisc is running,
if it is not running before check,
call `__qdisc_run`.


```c
  94 /* qdisc ->enqueue() return codes. */
  95 #define NET_XMIT_SUCCESS        0x00
  96 #define NET_XMIT_DROP           0x01    /* skb dropped                  */
  97 #define NET_XMIT_CN             0x02    /* congestion notification      */
  98 #define NET_XMIT_POLICED        0x03    /* skb is shot by police        */
  99 #define NET_XMIT_MASK           0x0f    /* qdisc flags in net/sch_generic.h */
 100
 101 /* NET_XMIT_CN is special. It does not guarantee that this packet is lost. It
 102  * indicates that the device will soon be dropping packets, or already drops
 103  * some packets of the same priority; prompting us to send less aggressively. */
 104 #define net_xmit_eval(e)        ((e) == NET_XMIT_CN ? 0 : (e))
 105 #define net_xmit_errno(e)       ((e) != NET_XMIT_CN ? -ENOBUFS : 0)
 106
 107 /* Driver transmit return codes */
 108 #define NETDEV_TX_MASK          0xf0
 109
 110 enum netdev_tx {
 111         __NETDEV_TX_MIN  = INT_MIN,     /* make sure enum is signed */
 112         NETDEV_TX_OK     = 0x00,        /* driver took care of packet */
 113         NETDEV_TX_BUSY   = 0x10,        /* driver tx path was busy*/
 114         NETDEV_TX_LOCKED = 0x20,        /* driver tx lock was already taken */
 115 };
 116 typedef enum netdev_tx netdev_tx_t;
```

#### How `__qdisc_run` works

`__qdisc_run` must be embraced by `qdisc_run_begin` and `qdisc_run_end`.
Before  `__qdisc_run`, set flag `__QDISC___STATE_RUNNING`. after run, remove it.
The flag and two functions ensure a qdisc will run only on a CPU at the smae time.

```c
194 void __qdisc_run(struct Qdisc *q)
195 {
196         int quota = weight_p;
197
198         while (qdisc_restart(q)) {
199                 /*
200                  * Ordered by possible occurrence: Postpone processing if
201                  * 1. we've exceeded packet quota
202                  * 2. another process needs the CPU;
203                  */
204                 if (--quota <= 0 || need_resched()) {
205                         __netif_schedule(q);
206                         break;
207                 }
208         }
209
210         qdisc_run_end(q);
211 }
```
`weight_p` is the max count of packets sent in **a** qdisc during a TX softirq.

If the qdisc has little packets, and they will be sent out in the while loop.
Else, the qdisc will be set state as `__QDISC_STATE_SCHED`,
and the qdisc will linked to `output_queue` of current cpu's `__get_cpu_var(softnet_data)`,
the TX softirq will be triggered to sent remain packet in the qdisc.

```c
156 /*
157  * NOTE: Called under qdisc_lock(q) with locally disabled BH.
158  *
159  * __QDISC_STATE_RUNNING guarantees only one CPU can process
160  * this qdisc at a time. qdisc_lock(q) serializes queue accesses for
161  * this queue.
162  *
163  *  netif_tx_lock serializes accesses to device driver.
164  *
165  *  qdisc_lock(q) and netif_tx_lock are mutually exclusive,
166  *  if one is grabbed, another must be free.
167  *
168  * Note, that this procedure can be called by a watchdog timer
169  *
170  * Returns to the caller:
171  *                              0  - queue is empty or throttled.
172  *                              >0 - queue is not empty.
173  *
174  */
175 static inline int qdisc_restart(struct Qdisc *q)
176 {
177         struct netdev_queue *txq;
178         struct net_device *dev;
179         spinlock_t *root_lock;
180         struct sk_buff *skb;
181
182         /* Dequeue packet */
183         skb = dequeue_skb(q);
184         if (unlikely(!skb))
185                 return 0;
186         WARN_ON_ONCE(skb_dst_is_noref(skb));
187         root_lock = qdisc_lock(q);
188         dev = qdisc_dev(q);
189         txq = netdev_get_tx_queue(dev, skb_get_queue_mapping(skb));
190
191         return sch_direct_xmit(skb, q, dev, txq, root_lock);
192 }
```

```c
109 /*
110  * Transmit one skb, and handle the return status as required. Holding the
111  * __QDISC_STATE_RUNNING bit guarantees that only one CPU can execute this
112  * function.
113  *
114  * Returns to the caller:
115  *                              0  - queue is empty or throttled.
116  *                              >0 - queue is not empty.
117  */
118 int sch_direct_xmit(struct sk_buff *skb, struct Qdisc *q,
119                     struct net_device *dev, struct netdev_queue *txq,
120                     spinlock_t *root_lock)
121 {
122         int ret = NETDEV_TX_BUSY;
123
124         /* And release qdisc */
125         spin_unlock(root_lock);
126
127         HARD_TX_LOCK(dev, txq, smp_processor_id());
128         if (!netif_xmit_frozen_or_stopped(txq))
129                 ret = dev_hard_start_xmit(skb, dev, txq);
130
131         HARD_TX_UNLOCK(dev, txq);
132
133         spin_lock(root_lock);
134
135         if (dev_xmit_complete(ret)) {
136                 /* Driver sent out skb successfully or skb was consumed */
137                 ret = qdisc_qlen(q);
138         } else if (ret == NETDEV_TX_LOCKED) {
139                 /* Driver try lock failed */
140                 ret = handle_dev_cpu_collision(skb, txq, q);
141         } else {
142                 /* Driver returned NETDEV_TX_BUSY - requeue skb */
143                 if (unlikely(ret != NETDEV_TX_BUSY))
144                         net_warn_ratelimited("BUG %s code %d qlen %d\n",
145                                              dev->name, ret, q->q.qlen);
146
147                 ret = dev_requeue_skb(skb, q);
148         }
149
150         if (ret && netif_xmit_frozen_or_stopped(txq))
151                 ret = 0;
152
153         return ret;
154 }
```

#### `skb->queue_mapping`
In multi-queue nic driver, it is used to indicate which queue is used to xmit packet.
It is set by `skb_set_queue_mapping` in `netdev_pick_tx`, `__dev_queue_xmit`.

#### busylock of `struct Qdisc`
As we said, Qdisc uses `__QDISC___STATE_RUNNING` to ensure,
for a same qdisc, **ONLY ONE** cpu xmit packet at the same time.
How to manage the other cpus?

`busylock` of `struct Qdisc` is used for this.
For a same qdisc, 
the first CPU, set the `__QDISC___STATE_RUNNING`.
the second CPU, grab the spinlock `busylock` of `struct Qdisc` 
for the third or more CPU, wait on spinlock `busylock` of `struct Qdisc`.

#### `qdisc_lock`
Where is `qdisc_lock` stored in the Qdisc.
```c
255 static inline spinlock_t *qdisc_lock(struct Qdisc *qdisc)
256 {
257         return &qdisc->q.lock;
258 }
``` 

```c
45 struct Qdisc {
...
85         struct sk_buff_head     q;
...
```

```c
 148 struct sk_buff_head {
 149         /* These two members must be first. */
 150         struct sk_buff  *next;
 151         struct sk_buff  *prev;
 152
 153         __u32           qlen;
 154         spinlock_t      lock;
 155 };
```

#### `NETIF_F_LLTX` 
`HARD_TX_LOCK` and `HARD_TX_UNLOCK` will embrance the driver's `ndo_start_xmit`.

```c
2805 #define HARD_TX_LOCK(dev, txq, cpu) {                   \
2806         if ((dev->features & NETIF_F_LLTX) == 0) {      \
2807                 __netif_tx_lock(txq, cpu);              \
2808         }                                               \
2809 }
2810
2811 #define HARD_TX_UNLOCK(dev, txq) {                      \
2812         if ((dev->features & NETIF_F_LLTX) == 0) {      \
2813                 __netif_tx_unlock(txq);                 \
2814         }                                               \
2815 }
```

For physical nic driver, it does nothing.
but for virutal nic device driver, it is used to ensure the packet sent in order.
```c
junwei@localhost:~/git/linux$ grep NETIF_F_LLTX net/ -Rw
net/ipv4/ipip.c:        dev->features           |= NETIF_F_LLTX;
net/ipv4/ip_gre.c:              dev->features |= NETIF_F_LLTX;
net/ipv4/ip_vti.c:      dev->features           |= NETIF_F_LLTX;
net/8021q/vlan_dev.c:   dev->features |= real_dev->vlan_features | NETIF_F_LLTX;
net/8021q/vlan_dev.c:   features |= NETIF_F_LLTX;
net/bridge/br_device.c: dev->features = COMMON_FEATURES | NETIF_F_LLTX | NETIF_F_NETNS_LOCAL |
net/l2tp/l2tp_eth.c:    dev->features           |= NETIF_F_LLTX;
net/ipv6/ip6_gre.c:             dev->features |= NETIF_F_LLTX;
net/ipv6/ip6_gre.c:             dev->features |= NETIF_F_LLTX;
net/ipv6/sit.c: dev->features           |= NETIF_F_LLTX;
net/hsr/hsr_device.c:   hsr_dev->features |= NETIF_F_LLTX;
net/openvswitch/vport-internal_dev.c:   netdev->features = NETIF_F_LLTX | NETIF_F_SG | NETIF_F_FRAGLIST |
net/openvswitch/vport-internal_dev.c:   netdev->hw_features = netdev->features & ~NETIF_F_LLTX;
```

```c
2710 static inline void __netif_tx_lock(struct netdev_queue *txq, int cpu)
2711 {
2712         spin_lock(&txq->_xmit_lock);
2713         txq->xmit_lock_owner = cpu;
2714 }
```

```c
2730 static inline void __netif_tx_unlock(struct netdev_queue *txq)
2731 {
2732         txq->xmit_lock_owner = -1;
2733         spin_unlock(&txq->_xmit_lock);
2734 }
```

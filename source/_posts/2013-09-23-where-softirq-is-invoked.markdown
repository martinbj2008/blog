---
layout: post
title: "where softirq is invoked"
date: 2013-09-23 10:36
comments: true
categories: [IRQs]
tags: [kernel, softirq]
---

##summary
softirq 真正干活的函数是`__do_softirq`。
linuxv3.11内核里能够执行`__do_softirq`,有如下调用,
这里指真正执行softirq的地方，不是触发(设置)softirq标志 !!!

1. 每个硬中断退出的时。
2. 当bh使能的时。
3. 发送回环报文时。

<!-- more -->

### case1 每个硬中断退出的时候。

#### call trace
```c
> irq_exit
> > invoke_softirq
> > > __do_softirq
```
#### `irq_exit`
```c
350 /*
351  * Exit an interrupt context. Process softirqs if needed and possible:
352  */
353 void irq_exit(void)
354 {
355 #ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
356         local_irq_disable();
357 #else
358         WARN_ON_ONCE(!irqs_disabled());
359 #endif
360
361         account_irq_exit_time(current);
362         trace_hardirq_exit();
363         sub_preempt_count(HARDIRQ_OFFSET);
364         if (!in_interrupt() && local_softirq_pending())
365                 invoke_softirq();
366
367         tick_irq_exit();
368         rcu_irq_exit();
369 }
```

#### `invoke_softirq`
```c
329 static inline void invoke_softirq(void)
330 {
331         if (!force_irqthreads)
332                 __do_softirq();
333         else
334                 wakeup_softirqd();
335 }
```


### case2 使能bh的时候

#### call trace
```c
> local_bh_enable
> > _local_bh_enable_ip
> > > do_softirq
> > > > (x86_64特有的)call_softirq
> > > > __do_softirq
```

#### `local_bh_enable`
```c
184 void local_bh_enable(void)
185 {
186         _local_bh_enable_ip(_RET_IP_);
187 }
188 EXPORT_SYMBOL(local_bh_enable);
```

#### `_local_bh_enable_ip`
NOTE: 使能bh的时候，刚开始抢占还是禁止的。只有软中断处理完了后，抢占才使能的。
顺便说一下，使能bh时，有可能发生进程切换(见`preempt_check_resched`).
```c
157 static inline void _local_bh_enable_ip(unsigned long ip)
158 {
159         WARN_ON_ONCE(in_irq() || irqs_disabled());
160 #ifdef CONFIG_TRACE_IRQFLAGS
161         local_irq_disable();
162 #endif
163         /*
164          * Are softirqs going to be turned on now:
165          */
166         if (softirq_count() == SOFTIRQ_DISABLE_OFFSET)
167                 trace_softirqs_on(ip);
168         /*
169          * Keep preemption disabled until we are done with
170          * softirq processing:
171          */
172         sub_preempt_count(SOFTIRQ_DISABLE_OFFSET - 1);
173 
174         if (unlikely(!in_interrupt() && local_softirq_pending()))
175                 do_softirq();
176    
177         dec_preempt_count();
178 #ifdef CONFIG_TRACE_IRQFLAGS
179         local_irq_enable();
180 #endif                  
181         preempt_check_resched();
182 }
```

#### `do_softirq`根据不同的体系机构有不同的版本。

当对应的硬件体系里没有定义`do_softirq`时，使用
`kernel/softirq.c`中通用的do_softirq`.
```c
286 #ifndef __ARCH_HAS_DO_SOFTIRQ
287 
288 asmlinkage void do_softirq(void)
289 {
290         __u32 pending;
291         unsigned long flags;
292 
293         if (in_interrupt())
294                 return;
295 
296         local_irq_save(flags);
297 
298         pending = local_softirq_pending();
299 
300         if (pending)
301                 __do_softirq();
302 
303         local_irq_restore(flags);
304 }
305 
306 #endif
```
而对于X86_64,有其自己的定义。
```c
 92 extern void call_softirq(void);
 93 
 94 asmlinkage void do_softirq(void)
 95 {
 96         __u32 pending;
 97         unsigned long flags;
 98 
 99         if (in_interrupt())
100                 return;
101 
102         local_irq_save(flags);
103         pending = local_softirq_pending();
104         /* Switch to interrupt stack */
105         if (pending) {
106                 call_softirq();
107                 WARN_ON_ONCE(softirq_count());
108         }
109         local_irq_restore(flags);
110 }
```
其中`call_softirq`是个用汇编语言写的一个函数定义在 arch/x86/kernel/entry_64.S

### `netif_rx_ni`

这一块的理解不是很透，仅以一个具体的例子来说明。`发送回环报文`.

#### 发送回环报文
#### call trace
```c
> dev_loopback_xmit
> > netif_rx_ni
> > do_softirq
> > > __do_softirq
```

loopback 报文会通过`netif_rx`函数，被重新放回到
per_cpu变量`softnet_data`下的队列`input_pkt_queue`里，
同时激发napi调用，进而激发软中断调用(只是设置标志位，不是真正调用）

另外，注意`netif_rx_ni`里没有禁止bh调度，而是只是禁止了抢占(`preempt_disable`).

##### `dev_loopback_xmit`

```c
2765 int dev_loopback_xmit(struct sk_buff *skb)
2766 {
2767         skb_reset_mac_header(skb);
2768         __skb_pull(skb, skb_network_offset(skb));
2769         skb->pkt_type = PACKET_LOOPBACK;
2770         skb->ip_summed = CHECKSUM_UNNECESSARY;
2771         WARN_ON(!skb_dst(skb));
2772         skb_dst_force(skb);
2773         netif_rx_ni(skb);
2774         return 0;
2775 }
```
##### `netif_rx_ni`
```c
3267 int netif_rx_ni(struct sk_buff *skb)
3268 {
3269         int err;
3270         
3271         preempt_disable();
3272         err = netif_rx(skb);
3273         if (local_softirq_pending())
3274                 do_softirq();
3275         preempt_enable();
3276         
3277         return err;
3278 }
3279 EXPORT_SYMBOL(netif_rx_ni);
```

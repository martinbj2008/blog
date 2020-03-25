---
layout: post
title: "register irq handler"
date: 2014-07-17 11:41
comments: true
categories: [IRQs]
tags: [irq]
---


### call trace
以`handle_level_irq`为例说明.
```
===> handle_level_irq
==> ==> handle_irq_event
==> ==> ==> handle_irq_event_percpu
==> ==> ==> ==>action->handler
```

#### where handler is registered
```
127 static inline int __must_check
128 request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
129             const char *name, void *dev)
130 {
131         return request_threaded_irq(irq, handler, NULL, flags, name, dev);
132 }
```
<!-- more -->

```
1473 int request_threaded_irq(unsigned int irq, irq_handler_t handler,
1474                          irq_handler_t thread_fn, unsigned long irqflags,
1475                          const char *devname, void *dev_id)
1476 {
1477         struct irqaction *action;
1478         struct irq_desc *desc;
1479         int retval;
1480
1481         /*
1482          * Sanity-check: shared interrupts must pass in a real dev-ID,
1483          * otherwise we'll have trouble later trying to figure out
1484          * which interrupt is which (messes up the interrupt freeing
1485          * logic etc).
1486          */
1487         if ((irqflags & IRQF_SHARED) && !dev_id)
1488                 return -EINVAL;
1489
1490         desc = irq_to_desc(irq);
1491         if (!desc)
1492                 return -EINVAL;
1493
1494         if (!irq_settings_can_request(desc) ||
1495             WARN_ON(irq_settings_is_per_cpu_devid(desc)))
1496                 return -EINVAL;
1497
1498         if (!handler) {
1499                 if (!thread_fn)
1500                         return -EINVAL;
1501                 handler = irq_default_primary_handler;
1502         }
1503
1504         action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
1505         if (!action)
1506                 return -ENOMEM;
1507
1508         action->handler = handler;
1509         action->thread_fn = thread_fn;
1510         action->flags = irqflags;
1511         action->name = devname;
1512         action->dev_id = dev_id;
1513
1514         chip_bus_lock(desc);
1515         retval = __setup_irq(irq, desc, action);
1516         chip_bus_sync_unlock(desc);
1517
1518         if (retval)
1519                 kfree(action);
1520
1521 #ifdef CONFIG_DEBUG_SHIRQ_FIXME
1522         if (!retval && (irqflags & IRQF_SHARED)) {
1523                 /*
1524                  * It's a shared IRQ -- the driver ought to be prepared for it
1525                  * to happen immediately, so let's make sure....
1526                  * We disable the irq to make sure that a 'real' IRQ doesn't
1527                  * run in parallel with our fake.
1528                  */
1529                 unsigned long flags;
1530
1531                 disable_irq(irq);
1532                 local_irq_save(flags);
1533
1534                 handler(irq, dev_id);
1535
1536                 local_irq_restore(flags);
1537                 enable_irq(irq);
1538         }
1539 #endif
1540         return retval;
1541 }
1542 EXPORT_SYMBOL(request_threaded_irq);

```
#### how to call the handler function
```
409 void
410 handle_level_irq(unsigned int irq, struct irq_desc *desc)
411 {
412         raw_spin_lock(&desc->lock);
413         mask_ack_irq(desc);
414
415         if (unlikely(irqd_irq_inprogress(&desc->irq_data)))
416                 if (!irq_check_poll(desc))
417                         goto out_unlock;
418
419         desc->istate &= ~(IRQS_REPLAY | IRQS_WAITING);
420         kstat_incr_irqs_this_cpu(irq, desc);
421
422         /*
423          * If its disabled or no action available
424          * keep it masked and get out of here
425          */
426         if (unlikely(!desc->action || irqd_irq_disabled(&desc->irq_data))) {
427                 desc->istate |= IRQS_PENDING;
428                 goto out_unlock;
429         }
430
431         handle_irq_event(desc);
432
433         cond_unmask_irq(desc);
434
435 out_unlock:
436         raw_spin_unlock(&desc->lock);
437 }
438 EXPORT_SYMBOL_GPL(handle_level_irq);
```

```
183 irqreturn_t handle_irq_event(struct irq_desc *desc)
184 {
185         struct irqaction *action = desc->action;
186         irqreturn_t ret;
187
188         desc->istate &= ~IRQS_PENDING;
189         irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);
190         raw_spin_unlock(&desc->lock);
191
192         ret = handle_irq_event_percpu(desc, action);
193
194         raw_spin_lock(&desc->lock);
195         irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);
196         return ret;
197 }
```

```
133 irqreturn_t
134 handle_irq_event_percpu(struct irq_desc *desc, struct irqaction *action)
135 {
136         irqreturn_t retval = IRQ_NONE;
137         unsigned int flags = 0, irq = desc->irq_data.irq;
138
139         do {
140                 irqreturn_t res;
141
142                 trace_irq_handler_entry(irq, action);
143                 res = action->handler(irq, action->dev_id);
144                 trace_irq_handler_exit(irq, action, res);
145
146                 if (WARN_ONCE(!irqs_disabled(),"irq %u handler %pF enabled interrupts\n",
147                               irq, action->handler))
148                         local_irq_disable();
149
150                 switch (res) {
151                 case IRQ_WAKE_THREAD:
152                         /*
153                          * Catch drivers which return WAKE_THREAD but
154                          * did not set up a thread function
155                          */
156                         if (unlikely(!action->thread_fn)) {
157                                 warn_no_thread(irq, action);
158                                 break;
159                         }
160
161                         __irq_wake_thread(desc, action);
162
163                         /* Fall through to add to randomness */
164                 case IRQ_HANDLED:
165                         flags |= action->flags;
166                         break;
167
168                 default:
169                         break;
170                 }
171
172                 retval |= res;
173                 action = action->next;
174         } while (action);
175
176         add_interrupt_randomness(irq, flags);
177
178         if (!noirqdebug)
179                 note_interrupt(irq, desc, retval);
180         return retval;
181 }
```

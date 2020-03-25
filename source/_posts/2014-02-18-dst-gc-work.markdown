---
layout: post
title: "Delayed work: dst_gc_work"
date: 2014-02-18 15:26
comments: true
categories: [IRQs]
tags: [workqueue]
---
### summary
A delayed work will first start a timer,
and when timeout, the delayed work will be put a `worker_pool`'s
`worklist` or a `pool_workqueue`'s `delayed_works`

### how to use delayed work
#### data structure
```c
113 struct delayed_work {
114         struct work_struct work;
115         struct timer_list timer;
116
117         /* target workqueue and CPU ->timer uses to queue ->work */
118         struct workqueue_struct *wq;
119         int cpu;
120 };
```
<!-- more -->

```c
157 #define __WORK_INITIALIZER(n, f) {                                      \
158         .data = WORK_DATA_STATIC_INIT(),                                \
159         .entry  = { &(n).entry, &(n).entry },                           \
160         .func = (f),                                                    \
161         __WORK_INIT_LOCKDEP_MAP(#n, &(n))                               \
162         }
163
164 #define __DELAYED_WORK_INITIALIZER(n, f, tflags) {                      \
165         .work = __WORK_INITIALIZER((n).work, (f)),                      \
166         .timer = __TIMER_INITIALIZER(delayed_work_timer_fn,             \
167                                      0, (unsigned long)&(n),            \
168                                      (tflags) | TIMER_IRQSAFE),         \
169         }
170
171 #define DECLARE_WORK(n, f)                                              \
172         struct work_struct n = __WORK_INITIALIZER(n, f)
173
174 #define DECLARE_DELAYED_WORK(n, f)                                      \
175         struct delayed_work n = __DELAYED_WORK_INITIALIZER(n, f, 0)
```

#### defination
```c
 52 static void dst_gc_task(struct work_struct *work);
 55 static DECLARE_DELAYED_WORK(dst_gc_work, dst_gc_task);
```

```c 
 63 static void dst_gc_task(struct work_struct *work)
 64 {
...
```

#### api to schedule workqueue.
`schedule_delayed_work` will put the work into a `struct work_pool`'s `worklist`
or a `pool_workqueue`'s `delayed_works`.

```c
586 /**
587  * schedule_delayed_work - put work task in global workqueue after delay
588  * @dwork: job to be done
589  * @delay: number of jiffies to wait or 0 for immediate execution
590  *
591  * After waiting for a given time this puts a job in the kernel-global
592  * workqueue.
593  */
594 static inline bool schedule_delayed_work(struct delayed_work *dwork,
595                                          unsigned long delay)
596 {
597         return queue_delayed_work(system_wq, dwork, delay);
598 }
```

```c
...
138                 schedule_delayed_work(&dst_gc_work, expires);
...
```

#### call trace
```c
> schedule_delayed_work
> > queue_delayed_work
> > > queue_delayed_work_on
> > > > __queue_delayed_work
> > > > > start a timer with timer function delayed_work_timer_fn
> > > > > > delayed_work_timer_fn
> > > > > > > __queue_work
```

#### core function `__queue_work`

```c
1314 static void __queue_work(int cpu, struct workqueue_struct *wq,
1315                          struct work_struct *work)
1316 {
1317         struct pool_workqueue *pwq;
1318         struct worker_pool *last_pool;
1319         struct list_head *worklist;
1320         unsigned int work_flags;
1321         unsigned int req_cpu = cpu;
1322
1323         /*
1324          * While a work item is PENDING && off queue, a task trying to
1325          * steal the PENDING will busy-loop waiting for it to either get
1326          * queued or lose PENDING.  Grabbing PENDING and queueing should
1327          * happen with IRQ disabled.
1328          */
1329         WARN_ON_ONCE(!irqs_disabled());
1330
1331         debug_work_activate(work);
1332
1333         /* if draining, only works from the same workqueue are allowed */
1334         if (unlikely(wq->flags & __WQ_DRAINING) &&
1335             WARN_ON_ONCE(!is_chained_work(wq)))
1336                 return;
1337 retry:
1338         if (req_cpu == WORK_CPU_UNBOUND)
1339                 cpu = raw_smp_processor_id();
1340
1341         /* pwq which will be used unless @work is executing elsewhere */
1342         if (!(wq->flags & WQ_UNBOUND))
1343                 pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
1344         else
1345                 pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
1346
1347         /*
1348          * If @work was previously on a different pool, it might still be
1349          * running there, in which case the work needs to be queued on that
1350          * pool to guarantee non-reentrancy.
1351          */
1352         last_pool = get_work_pool(work);
1353         if (last_pool && last_pool != pwq->pool) {
1354                 struct worker *worker;
1355
1356                 spin_lock(&last_pool->lock);
1357
1358                 worker = find_worker_executing_work(last_pool, work);
1359
1360                 if (worker && worker->current_pwq->wq == wq) {
1361                         pwq = worker->current_pwq;
1362                 } else {
1363                         /* meh... not running there, queue here */
1364                         spin_unlock(&last_pool->lock);
1365                         spin_lock(&pwq->pool->lock);
1366                 }
1367         } else {
1368                 spin_lock(&pwq->pool->lock);
1369         }
1370
1371         /*
1372          * pwq is determined and locked.  For unbound pools, we could have
1373          * raced with pwq release and it could already be dead.  If its
1374          * refcnt is zero, repeat pwq selection.  Note that pwqs never die
1375          * without another pwq replacing it in the numa_pwq_tbl or while
1376          * work items are executing on it, so the retrying is guaranteed to
1377          * make forward-progress.
1378          */
1379         if (unlikely(!pwq->refcnt)) {
1380                 if (wq->flags & WQ_UNBOUND) {
1381                         spin_unlock(&pwq->pool->lock);
1382                         cpu_relax();
1383                         goto retry;
1384                 }
1385                 /* oops */
1386                 WARN_ONCE(true, "workqueue: per-cpu pwq for %s on cpu%d has 0 refcnt",
1387                           wq->name, cpu);
1388         }
1389
1390         /* pwq determined, queue */
1391         trace_workqueue_queue_work(req_cpu, pwq, work);
1392
1393         if (WARN_ON(!list_empty(&work->entry))) {
1394                 spin_unlock(&pwq->pool->lock);
1395                 return;
1396         }
1397
1398         pwq->nr_in_flight[pwq->work_color]++;
1399         work_flags = work_color_to_flags(pwq->work_color);
1400
1401         if (likely(pwq->nr_active < pwq->max_active)) {
1402                 trace_workqueue_activate_work(work);
1403                 pwq->nr_active++;
1404                 worklist = &pwq->pool->worklist;
1405         } else {
1406                 work_flags |= WORK_STRUCT_DELAYED;
1407                 worklist = &pwq->delayed_works;
1408         }
1409
1410         insert_work(pwq, work, worklist, work_flags);
1411
1412         spin_unlock(&pwq->pool->lock);
1413 }
```

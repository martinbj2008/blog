---
layout: post
title: "struct worker_pool->nr_running"
date: 2014-02-18 15:15
comments: true
categories: [IRQs]
tags:	[workqueue]
---

### Defination
```c
 141 /* struct worker is defined in workqueue_internal.h */
 142
 143 struct worker_pool {
 ...
 173         /*
 174          * The current concurrency level.  As it's likely to be accessed
 175          * from other CPUs during try_to_wake_up(), put it in a separate
 176          * cacheline.
 177          */
 178         atomic_t                nr_running ____cacheline_aligned_in_smp;
```

### Increase
```c
 813 /**
 814  * wq_worker_waking_up - a worker is waking up
 815  * @task: task waking up
 816  * @cpu: CPU @task is waking up to
 817  *
 818  * This function is called during try_to_wake_up() when a worker is
 819  * being awoken.
 820  *
 821  * CONTEXT:
 822  * spin_lock_irq(rq->lock)
 823  */
 824 void wq_worker_waking_up(struct task_struct *task, int cpu)
 825 {
 826         struct worker *worker = kthread_data(task);
 827
 828         if (!(worker->flags & WORKER_NOT_RUNNING)) {
 829                 WARN_ON_ONCE(worker->pool->cpu != cpu);
 830                 atomic_inc(&worker->pool->nr_running);
 831         }
 832 }
```

```c
 923 /**
 924  * worker_clr_flags - clear worker flags and adjust nr_running accordingly
 925  * @worker: self
 926  * @flags: flags to clear
 927  *
 928  * Clear @flags in @worker->flags and adjust nr_running accordingly.
 929  *
 930  * CONTEXT:
 931  * spin_lock_irq(pool->lock)
 932  */
 933 static inline void worker_clr_flags(struct worker *worker, unsigned int flags)
 934 {
 935         struct worker_pool *pool = worker->pool;
 936         unsigned int oflags = worker->flags;
 937
 938         WARN_ON_ONCE(worker->task != current);
 939
 940         worker->flags &= ~flags;
 941
 942         /*
 943          * If transitioning out of NOT_RUNNING, increment nr_running.  Note
 944          * that the nested NOT_RUNNING is not a noop.  NOT_RUNNING is mask
 945          * of multiple flags, not a single flag.
 946          */
 947         if ((flags & WORKER_NOT_RUNNING) && (oflags & WORKER_NOT_RUNNING))
 948                 if (!(worker->flags & WORKER_NOT_RUNNING))
 949                         atomic_inc(&pool->nr_running);
 950 }
```
### Decrease
```c
 885 /**
 886  * worker_set_flags - set worker flags and adjust nr_running accordingly
 887  * @worker: self
 888  * @flags: flags to set
 889  * @wakeup: wakeup an idle worker if necessary
 890  *
 891  * Set @flags in @worker->flags and adjust nr_running accordingly.  If
 892  * nr_running becomes zero and @wakeup is %true, an idle worker is
 893  * woken up.
 894  *
 895  * CONTEXT:
 896  * spin_lock_irq(pool->lock)
 897  */
 898 static inline void worker_set_flags(struct worker *worker, unsigned int flags,
 899                                     bool wakeup)
 900 {
 901         struct worker_pool *pool = worker->pool;
 902
 903         WARN_ON_ONCE(worker->task != current);
 904
 905         /*
 906          * If transitioning into NOT_RUNNING, adjust nr_running and
 907          * wake up an idle worker as necessary if requested by
 908          * @wakeup.
 909          */
 910         if ((flags & WORKER_NOT_RUNNING) &&
 911             !(worker->flags & WORKER_NOT_RUNNING)) {
 912                 if (wakeup) {
 913                         if (atomic_dec_and_test(&pool->nr_running) &&
 914                             !list_empty(&pool->worklist))
 915                                 wake_up_worker(pool);
 916                 } else
 917                         atomic_dec(&pool->nr_running);
 918         }
 919
 920         worker->flags |= flags;
 921 }
```

```c
 834 /**
 835  * wq_worker_sleeping - a worker is going to sleep
 836  * @task: task going to sleep
 837  * @cpu: CPU in question, must be the current CPU number
 838  *
 839  * This function is called during schedule() when a busy worker is
 840  * going to sleep.  Worker on the same cpu can be woken up by
 841  * returning pointer to its task.
 842  *
 843  * CONTEXT:
 844  * spin_lock_irq(rq->lock)
 845  *
 846  * Return:
 847  * Worker task on @cpu to wake up, %NULL if none.
 848  */
 849 struct task_struct *wq_worker_sleeping(struct task_struct *task, int cpu)
 850 {
 851         struct worker *worker = kthread_data(task), *to_wakeup = NULL;
 852         struct worker_pool *pool;
 853
 854         /*
 855          * Rescuers, which may not have all the fields set up like normal
 856          * workers, also reach here, let's not access anything before
 857          * checking NOT_RUNNING.
 858          */
 859         if (worker->flags & WORKER_NOT_RUNNING)
 860                 return NULL;
 861
 862         pool = worker->pool;
 863
 864         /* this can only happen on the local cpu */
 865         if (WARN_ON_ONCE(cpu != raw_smp_processor_id()))
 866                 return NULL;
 867
 868         /*
 869          * The counterpart of the following dec_and_test, implied mb,
 870          * worklist not empty test sequence is in insert_work().
 871          * Please read comment there.
 872          *
 873          * NOT_RUNNING is clear.  This means that we're bound to and
 874          * running on the local cpu w/ rq lock held and preemption
 875          * disabled, which in turn means that none else could be
 876          * manipulating idle_list, so dereferencing idle_list without pool
 877          * lock is safe.
 878          */
 879         if (atomic_dec_and_test(&pool->nr_running) &&
 880             !list_empty(&pool->worklist))
 881                 to_wakeup = first_worker(pool);
 882         return to_wakeup ? to_wakeup->task : NULL;
 883 }
```

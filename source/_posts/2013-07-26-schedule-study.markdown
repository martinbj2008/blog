---
layout: post
title: "draft: struct rq"
date: 2013-07-26 08:10
comments: true
categories: [sched]
tags: [kernel, sched]
---

##summary:

1. struct rq is basic data structure.
2. there is a realtime process(es) and normal process(es) are stored in 
different  sub-runqueue `*_rq` of rq? 
```c
426         struct cfs_rq cfs;
427         struct rt_rq rt;
```
<!-- more -->

```c
 393 /*
 394  * This is the main, per-CPU runqueue data structure.
 395  *
 396  * Locking rule: those places that want to lock multiple runqueues
 397  * (such as the load balancing or the thread migration code), lock
 398  * acquire operations must be ordered by ascending &runqueue.
 399  */
 400 struct rq {
 401         /* runqueue lock: */
 402         raw_spinlock_t lock;
 403 
 404         /*
 405          * nr_running and cpu_load should be in the same cacheline because
 406          * remote CPUs use both these fields when doing load calculation.
 407          */
 408         unsigned int nr_running;
 409         #define CPU_LOAD_IDX_MAX 5
 410         unsigned long cpu_load[CPU_LOAD_IDX_MAX];
 411         unsigned long last_load_update_tick;
 412 #ifdef CONFIG_NO_HZ_COMMON
 413         u64 nohz_stamp;
 414         unsigned long nohz_flags;
 415 #endif
 416 #ifdef CONFIG_NO_HZ_FULL
 417         unsigned long last_sched_tick;
 418 #endif
 419         int skip_clock_update;
 420 
 421         /* capture load from *all* tasks on this cpu: */
 422         struct load_weight load;
 423         unsigned long nr_load_updates;
 424         u64 nr_switches;
 425 
 426         struct cfs_rq cfs;
 427         struct rt_rq rt;
 428 
 429 #ifdef CONFIG_FAIR_GROUP_SCHED
 430         /* list of leaf cfs_rq on this cpu: */
 431         struct list_head leaf_cfs_rq_list;
 432 #ifdef CONFIG_SMP
 433         unsigned long h_load_throttle;
 434 #endif /* CONFIG_SMP */
 435 #endif /* CONFIG_FAIR_GROUP_SCHED */
 436 
 437 #ifdef CONFIG_RT_GROUP_SCHED
 438         struct list_head leaf_rt_rq_list;
 439 #endif
 440 
 441         /*
 442          * This is part of a global counter where only the total sum
 443          * over all CPUs matters. A task can increase this counter on
 444          * one CPU and if it got migrated afterwards it may decrease
 445          * it on another CPU. Always updated under the runqueue lock:
 446          */
 447         unsigned long nr_uninterruptible;
 448 
 449         struct task_struct *curr, *idle, *stop;
 450         unsigned long next_balance;
 451         struct mm_struct *prev_mm;
 452 
 453         u64 clock;
 454         u64 clock_task;
 455 
 456         atomic_t nr_iowait;
 457 
 458 #ifdef CONFIG_SMP
 459         struct root_domain *rd;
 460         struct sched_domain *sd;
 461 
 462         unsigned long cpu_power;
 463 
 464         unsigned char idle_balance;
 465         /* For active balancing */
 466         int post_schedule;
 467         int active_balance;
 468         int push_cpu;
 469         struct cpu_stop_work active_balance_work;
 470         /* cpu of this runqueue: */
 471         int cpu;
 472         int online;
 473 
 474         struct list_head cfs_tasks;
 475 
 476         u64 rt_avg;
 477         u64 age_stamp;
 478         u64 idle_stamp;
 479         u64 avg_idle;
 480 #endif
 481 
 482 #ifdef CONFIG_IRQ_TIME_ACCOUNTING
 483         u64 prev_irq_time;
 484 #endif
 485 #ifdef CONFIG_PARAVIRT
 486         u64 prev_steal_time;
 487 #endif
 488 #ifdef CONFIG_PARAVIRT_TIME_ACCOUNTING
 489         u64 prev_steal_time_rq;
 490 #endif
 491 
 492         /* calc_load related fields */
 493         unsigned long calc_load_update;
 494         long calc_load_active;
 495 
 496 #ifdef CONFIG_SCHED_HRTICK
 497 #ifdef CONFIG_SMP
 498         int hrtick_csd_pending;
 499         struct call_single_data hrtick_csd;
 500 #endif
 501         struct hrtimer hrtick_timer;
 502 #endif
 503 
 504 #ifdef CONFIG_SCHEDSTATS
 505         /* latency stats */
 506         struct sched_info rq_sched_info;
 507         unsigned long long rq_cpu_time;
 508         /* could above be rq->cfs_rq.exec_clock + rq->rt_rq.rt_runtime ? */
 509 
 510         /* sys_sched_yield() stats */
 511         unsigned int yld_count;
 512 
 513         /* schedule() stats */
 514         unsigned int sched_count;
 515         unsigned int sched_goidle;
 516 
 517         /* try_to_wake_up() stats */
 518         unsigned int ttwu_count;
 519         unsigned int ttwu_local;
 520 #endif
 521 
 522 #ifdef CONFIG_SMP
 523         struct llist_head wake_list;
 524 #endif
 525 
 526         struct sched_avg avg;
 527 };

```

---
layout: post
title: "draft: how to select a realtime task when schedule"
date: 2013-07-26 09:00
comments: true
categories: [sched]
tags: [kernel, sched]
---

##summary

There is a `struct rt_rq` in `struct rq`. `struct rt_rq` is used to store all
the realtime task in current `struct rq`. which has a `struct rt_prio_array active`.

<!-- more -->

```c
struct rq {
	...
	struct rt_rq {
		...
		struct rt_prio_array active {
			DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1); /* include 1 bit for delimiter */
			struct list_head queue[MAX_RT_PRIO];
		}
		...
	}
	...
}
```

*for a* `rq`:
All the realtime tasks are stored by a bit map and a list array.

1. `DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1)`
   This is a bit map to show if a priority level has task to be selected.
   One bit set to 1 means one or more task(s).

2. `struct list_head queue[MAX_RT_PRIO]`
   This is list array, each element is a list head.
   All tasks are linked into different lists according their priority by 
  `struct sched_rt_entity`'s `struct list_head run_list`.

```c
1001 struct sched_rt_entity {
1002         struct list_head run_list
```
   Each list has a corresponding bit in previous bitmap.

When we need pick a task, we find find the first bit in the map,
and then select one task from the corresponding list.

Q1: what is group_rt_rq? what is it used for?
  CONFIG_RT_GROUP_SCHED??



### Call Trac
```c
> pick_next_task
> > for_each_class(class) {
        p = class->pick_next_task(rq);
> > > pick_next_task_rt
> > > > _pick_next_task_rt
> > > > > pick_next_rt_entity
> > > > > > sched_find_first_bit(array->bitmap);
> > > > > > > list_entry(queue->next, struct sched_rt_entity, run_list);
```

##Data Structure


### realtime runqueue `struct rt_rq`

```c
 332 /* Real-Time classes' related field in a runqueue: */
 333 struct rt_rq {
 334         struct rt_prio_array active;
 335         unsigned int rt_nr_running;
 336 #if defined CONFIG_SMP || defined CONFIG_RT_GROUP_SCHED
 337         struct {
 338                 int curr; /* highest queued rt task prio */
 339 #ifdef CONFIG_SMP
 340                 int next; /* next highest */
 341 #endif
 342         } highest_prio;
 343 #endif
 344 #ifdef CONFIG_SMP
 345         unsigned long rt_nr_migratory;
 346         unsigned long rt_nr_total;
 347         int overloaded;
 348         struct plist_head pushable_tasks;
 349 #endif
 350         int rt_throttled;
 351         u64 rt_time;
 352         u64 rt_runtime;
 353         /* Nests inside the rq lock: */
 354         raw_spinlock_t rt_runtime_lock;
 355 
 356 #ifdef CONFIG_RT_GROUP_SCHED
 357         unsigned long rt_nr_boosted;
 358 
 359         struct rq *rq;
 360         struct task_group *tg;
 361 #endif
 362 };
 363 
```

### `struct rt_prio_array`
```c
  95 /*
  96  * This is the priority-queue data structure of the RT scheduling class:
  97  */
  98 struct rt_prio_array {
  99         DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1); /* include 1 bit for delimiter */
 100         struct list_head queue[MAX_RT_PRIO];
 101 };
```

```c
1001 struct sched_rt_entity {
1002         struct list_head run_list;
1003         unsigned long timeout;  
1004         unsigned long watchdog_stamp;
1005         unsigned int time_slice;
1006 
1007         struct sched_rt_entity *back;
1008 #ifdef CONFIG_RT_GROUP_SCHED
1009         struct sched_rt_entity  *parent;
1010         /* rq on which this entity is (to be) queued: */
1011         struct rt_rq            *rt_rq;
1012         /* rq "owned" by this entity/group: */
1013         struct rt_rq            *my_q;
1014 #endif
1015 };
```


### `pick_next_task_rt`

```c
1323 static struct task_struct *pick_next_task_rt(struct rq *rq)
1324 {
1325         struct task_struct *p = _pick_next_task_rt(rq);
1326 
1327         /* The running task is never eligible for pushing */
1328         if (p)
1329                 dequeue_pushable_task(rq, p);
1330 
1331 #ifdef CONFIG_SMP
1332         /*
1333          * We detect this state here so that we can avoid taking the RQ
1334          * lock again later if there is no need to push
1335          */
1336         rq->post_schedule = has_pushable_tasks(rq);
1337 #endif
1338 
1339         return p;
1340 }
```


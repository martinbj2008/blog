---
layout: post
title: "draft: how to pick next task"
date: 2013-07-26 08:23
comments: true
categories: [sched]
tags: [kernel, sched]
---

## summary

There are four `sched_class`, 
`stop_sched_class --> rt_sched_class --> fair_sched_class --> idle_sched_class`

They are linked one by one staticly by `struct sched_class->next` by their defination.

Each `sched_class` has method `pick_next_task`, which is used to select
a perfect process to run from each `sched_class`'s runqueue.

When we need schedule, the four `pick_next_task` will be called one by one.

As a optimization, most time there is no rt task in running state,
in this case we can directly call `fair_sched_class`.

<!-- more -->

```c
2322         /*
2323          * Optimization: we know that if all tasks are in
2324          * the fair class we can call that function directly:
2325          */
2326         if (likely(rq->nr_running == rq->cfs.h_nr_running)) {
2327                 p = fair_sched_class.pick_next_task(rq);
```

question:
1. how `stop_class` and `idle_class`, they has no task? idle_task?

## data structure


### `sched_class`

```c
1009 #define sched_class_highest (&stop_sched_class)
1010 #define for_each_class(class) \
1011    for (class = sched_class_highest; class; class = class->next)
1012 
1013 extern const struct sched_class stop_sched_class;
1014 extern const struct sched_class rt_sched_class;
1015 extern const struct sched_class fair_sched_class;
1016 extern const struct sched_class idle_sched_class;
```

```c
 963 struct sched_class { 
 964         const struct sched_class *next;
 965         
 966         void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
 967         void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
 968         void (*yield_task) (struct rq *rq);
 969         bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);
 970 
 971         void (*check_preempt_curr) (struct rq *rq, struct task_struct *p, int flags);
 972         
 973         struct task_struct * (*pick_next_task) (struct rq *rq);
 974         void (*put_prev_task) (struct rq *rq, struct task_struct *p);
 975         
 976 #ifdef CONFIG_SMP
 977         int  (*select_task_rq)(struct task_struct *p, int sd_flag, int flags);
 978         void (*migrate_task_rq)(struct task_struct *p, int next_cpu);
 979 
 980         void (*pre_schedule) (struct rq *this_rq, struct task_struct *task);
 981         void (*post_schedule) (struct rq *this_rq);
 982         void (*task_waking) (struct task_struct *task);
 983         void (*task_woken) (struct rq *this_rq, struct task_struct *task);
 984 
 985         void (*set_cpus_allowed)(struct task_struct *p,
 986                                  const struct cpumask *newmask);
 987 
 988         void (*rq_online)(struct rq *rq);
 989         void (*rq_offline)(struct rq *rq);
 990 #endif
 991 
 992         void (*set_curr_task) (struct rq *rq);
 993         void (*task_tick) (struct rq *rq, struct task_struct *p, int queued);
 994         void (*task_fork) (struct task_struct *p);
 995 
 996         void (*switched_from) (struct rq *this_rq, struct task_struct *task);
 997         void (*switched_to) (struct rq *this_rq, struct task_struct *task);
 998         void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
 999                              int oldprio);
1000 
1001         unsigned int (*get_rr_interval) (struct rq *rq,
1002                                          struct task_struct *task);
1003 
1004 #ifdef CONFIG_FAIR_GROUP_SCHED
1005         void (*task_move_group) (struct task_struct *p, int on_rq);
1006 #endif
1007 };
```

### `stop_sched_class`
```c
102 /*
103  * Simple, special scheduling class for the per-CPU stop tasks:
104  */
105 const struct sched_class stop_sched_class = {
106         .next                   = &rt_sched_class,
107 
108         .enqueue_task           = enqueue_task_stop,
109         .dequeue_task           = dequeue_task_stop,
110         .yield_task             = yield_task_stop,
111 
112         .check_preempt_curr     = check_preempt_curr_stop,
113 
114         .pick_next_task         = pick_next_task_stop,
115         .put_prev_task          = put_prev_task_stop,
116 
117 #ifdef CONFIG_SMP
118         .select_task_rq         = select_task_rq_stop,
119 #endif
120 
121         .set_curr_task          = set_curr_task_stop,
122         .task_tick              = task_tick_stop,
123 
124         .get_rr_interval        = get_rr_interval_stop,
125 
126         .prio_changed           = prio_changed_stop,
127         .switched_to            = switched_to_stop,
128 };
```

### `rt_sched_class`

```c
1967 const struct sched_class rt_sched_class = {
1968         .next                   = &fair_sched_class,
1969         .enqueue_task           = enqueue_task_rt,
1970         .dequeue_task           = dequeue_task_rt,
1971         .yield_task             = yield_task_rt,
1972 
1973         .check_preempt_curr     = check_preempt_curr_rt,
1974 
1975         .pick_next_task         = pick_next_task_rt,
1976         .put_prev_task          = put_prev_task_rt,
1977 
1978 #ifdef CONFIG_SMP
1979         .select_task_rq         = select_task_rq_rt,
1980        
1981         .set_cpus_allowed       = set_cpus_allowed_rt,
1982         .rq_online              = rq_online_rt,
1983         .rq_offline             = rq_offline_rt,
1984         .pre_schedule           = pre_schedule_rt,
1985         .post_schedule          = post_schedule_rt,
1986         .task_woken             = task_woken_rt,
1987         .switched_from          = switched_from_rt,
1988 #endif 
1989        
1990         .set_curr_task          = set_curr_task_rt,
1991         .task_tick              = task_tick_rt,
1992        
1993         .get_rr_interval        = get_rr_interval_rt,
1994 
1995         .prio_changed           = prio_changed_rt,
1996         .switched_to            = switched_to_rt,
1997 };
```

### `fair_sched_class`

```c
6170 /*      
6171  * All the scheduling class methods:
6172  */     
6173 const struct sched_class fair_sched_class = {
6174         .next                   = &idle_sched_class,
6175         .enqueue_task           = enqueue_task_fair,
6176         .dequeue_task           = dequeue_task_fair,
6177         .yield_task             = yield_task_fair,
6178         .yield_to_task          = yield_to_task_fair,
6179        
6180         .check_preempt_curr     = check_preempt_wakeup,
6181         
6182         .pick_next_task         = pick_next_task_fair,
6183         .put_prev_task          = put_prev_task_fair,
6184 
6185 #ifdef CONFIG_SMP
6186         .select_task_rq         = select_task_rq_fair,
6187         .migrate_task_rq        = migrate_task_rq_fair,
6188        
6189         .rq_online              = rq_online_fair,
6190         .rq_offline             = rq_offline_fair,
6191        
6192         .task_waking            = task_waking_fair,
6193 #endif
6194         
6195         .set_curr_task          = set_curr_task_fair,
6196         .task_tick              = task_tick_fair,
6197         .task_fork              = task_fork_fair,
6198 
6199         .prio_changed           = prio_changed_fair, 
6200         .switched_from          = switched_from_fair,
6201         .switched_to            = switched_to_fair,
6202 
6203         .get_rr_interval        = get_rr_interval_fair,
6204 
6205 #ifdef CONFIG_FAIR_GROUP_SCHED
6206         .task_move_group        = task_move_group_fair,
6207 #endif
6208 };
6209 
```

### `idle_sched_class`

```c
 87 /*       
 88  * Simple, special scheduling class for the per-CPU idle tasks:
 89  */
 90 const struct sched_class idle_sched_class = {
 91         /* .next is NULL */
 92         /* no enqueue/yield_task for idle tasks */
 93 
 94         /* dequeue is not valid, we print a debug message there: */
 95         .dequeue_task           = dequeue_task_idle,
 96 
 97         .check_preempt_curr     = check_preempt_curr_idle,
 98 
 99         .pick_next_task         = pick_next_task_idle,
100         .put_prev_task          = put_prev_task_idle,
101 
102 #ifdef CONFIG_SMP
103         .select_task_rq         = select_task_rq_idle,
104         .pre_schedule           = pre_schedule_idle,
105         .post_schedule          = post_schedule_idle,
106 #endif   
107 
108         .set_curr_task          = set_curr_task_idle,
109         .task_tick              = task_tick_idle,
110 
111         .get_rr_interval        = get_rr_interval_idle,
112 
113         .prio_changed           = prio_changed_idle,
114         .switched_to            = switched_to_idle,
115 };
```
### `pick_next_task`

```c
2313 /*
2314  * Pick up the highest-prio task:
2315  */
2316 static inline struct task_struct *
2317 pick_next_task(struct rq *rq)
2318 { 
2319         const struct sched_class *class;
2320         struct task_struct *p;
2321   
2322         /*
2323          * Optimization: we know that if all tasks are in
2324          * the fair class we can call that function directly:
2325          */
2326         if (likely(rq->nr_running == rq->cfs.h_nr_running)) {
2327                 p = fair_sched_class.pick_next_task(rq); 
2328                 if (likely(p))     
2329                         return p;  
2330         }
2331   
2332         for_each_class(class) {    
2333                 p = class->pick_next_task(rq);
2334                 if (p)             
2335                         return p;  
2336         }
2337   
2338         BUG(); /* the idle class will always have a runnable task */
2339 } 
```


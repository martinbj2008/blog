---
layout: post
title: "workqueue basic structure"
date: 2014-02-12 18:19
comments: true
categories: [IRQs]
tags: [workqueue, bh]
---

### summary
<!-- more-->

####`worker`
```c
 15 /*
 16  * The poor guys doing the actual heavy lifting.  All on-duty workers are
 17  * either serving the manager role, on idle list or on busy hash.  For
 18  * details on the locking annotation (L, I, X...), refer to workqueue.c.
 19  *
 20  * Only to be used in workqueue and async.
 21  */
 22 struct worker {
 23         /* on idle list while idle, on busy hash table while busy */
 24         union {
 25                 struct list_head        entry;  /* L: while idle */
 26                 struct hlist_node       hentry; /* L: while busy */
 27         };
 28
 29         struct work_struct      *current_work;  /* L: work being processed */
 30         work_func_t             current_func;   /* L: current_work's fn */
 31         struct pool_workqueue   *current_pwq; /* L: current_work's pwq */
 32         bool                    desc_valid;     /* ->desc is valid */
 33         struct list_head        scheduled;      /* L: scheduled works */
 34
 35         /* 64 bytes boundary on 64bit, 32 on 32bit */
 36
 37         struct task_struct      *task;          /* I: worker task */
 38         struct worker_pool      *pool;          /* I: the associated pool */
 39                                                 /* L: for rescuers */
 40
 41         unsigned long           last_active;    /* L: last active timestamp */
 42         unsigned int            flags;          /* X: flags */
 43         int                     id;             /* I: worker id */
 44
 45         /*
 46          * Opaque string set with work_set_desc().  Printed out with task
 47          * dump for debugging - WARN, BUG, panic or sysrq.
 48          */
 49         char                    desc[WORKER_DESC_LEN];
 50
 51         /* used only by rescuers to point to the target workqueue */
 52         struct workqueue_struct *rescue_wq;     /* I: the workqueue to rescue */
 53 };
```
#### `workqueue_struct`
```c
 228 /*
 229  * The externally visible workqueue.  It relays the issued work items to
 230  * the appropriate worker_pool through its pool_workqueues.
 231  */
 232 struct workqueue_struct {
 233         struct list_head        pwqs;           /* WR: all pwqs of this wq */
 234         struct list_head        list;           /* PL: list of all workqueues */
 235
 236         struct mutex            mutex;          /* protects this wq */
 237         int                     work_color;     /* WQ: current work color */
 238         int                     flush_color;    /* WQ: current flush color */
 239         atomic_t                nr_pwqs_to_flush; /* flush in progress */
 240         struct wq_flusher       *first_flusher; /* WQ: first flusher */
 241         struct list_head        flusher_queue;  /* WQ: flush waiters */
 242         struct list_head        flusher_overflow; /* WQ: flush overflow list */
 243
 244         struct list_head        maydays;        /* MD: pwqs requesting rescue */
 245         struct worker           *rescuer;       /* I: rescue worker */
 246
 247         int                     nr_drainers;    /* WQ: drain in progress */
 248         int                     saved_max_active; /* WQ: saved pwq max_active */
 249
 250         struct workqueue_attrs  *unbound_attrs; /* WQ: only for unbound wqs */
 251         struct pool_workqueue   *dfl_pwq;       /* WQ: only for unbound wqs */
 252
 253 #ifdef CONFIG_SYSFS
 254         struct wq_device        *wq_dev;        /* I: for sysfs interface */
 255 #endif
 256 #ifdef CONFIG_LOCKDEP
 257         struct lockdep_map      lockdep_map;
 258 #endif
 259         char                    name[WQ_NAME_LEN]; /* I: workqueue name */
 260
 261         /* hot fields used during command issue, aligned to cacheline */
 262         unsigned int            flags ____cacheline_aligned; /* WQ: WQ_* flags */
 263         struct pool_workqueue __percpu *cpu_pwqs; /* I: per-cpu pwqs */
 264         struct pool_workqueue __rcu *numa_pwq_tbl[]; /* FR: unbound pwqs indexed by node */
 265 };
```

#### `struct worker_pool`
```c
 143 struct worker_pool {
 144         spinlock_t              lock;           /* the pool lock */
 145         int                     cpu;            /* I: the associated cpu */
 146         int                     node;           /* I: the associated node ID */
 147         int                     id;             /* I: pool ID */
 148         unsigned int            flags;          /* X: flags */
 149
 150         struct list_head        worklist;       /* L: list of pending works */
 151         int                     nr_workers;     /* L: total number of workers */
 152
 153         /* nr_idle includes the ones off idle_list for rebinding */
 154         int                     nr_idle;        /* L: currently idle ones */
 155
 156         struct list_head        idle_list;      /* X: list of idle workers */
 157         struct timer_list       idle_timer;     /* L: worker idle timeout */
 158         struct timer_list       mayday_timer;   /* L: SOS timer for workers */
 159
 160         /* a workers is either on busy_hash or idle_list, or the manager */
 161         DECLARE_HASHTABLE(busy_hash, BUSY_WORKER_HASH_ORDER);
 162                                                 /* L: hash of busy workers */
 163
 164         /* see manage_workers() for details on the two manager mutexes */
 165         struct mutex            manager_arb;    /* manager arbitration */
 166         struct mutex            manager_mutex;  /* manager exclusion */
 167         struct idr              worker_idr;     /* MG: worker IDs and iteration */
 168
 169         struct workqueue_attrs  *attrs;         /* I: worker attributes */
 170         struct hlist_node       hash_node;      /* PL: unbound_pool_hash node */
 171         int                     refcnt;         /* PL: refcnt for unbound pools */
 172
 173         /*
 174          * The current concurrency level.  As it's likely to be accessed
 175          * from other CPUs during try_to_wake_up(), put it in a separate
 176          * cacheline.
 177          */
 178         atomic_t                nr_running ____cacheline_aligned_in_smp;
 179
 180         /*
 181          * Destruction of pool is sched-RCU protected to allow dereferences
 182          * from get_work_pool().
 183          */
 184         struct rcu_head         rcu;
 185 } ____cacheline_aligned_in_smp;
```

```c
297 static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS],
298                                      cpu_worker_pools);
```

#### `struct pool_workqueue`
```c
 193 struct pool_workqueue {
 194         struct worker_pool      *pool;          /* I: the associated pool */
 195         struct workqueue_struct *wq;            /* I: the owning workqueue */
 196         int                     work_color;     /* L: current color */
 197         int                     flush_color;    /* L: flushing color */
 198         int                     refcnt;         /* L: reference count */
 199         int                     nr_in_flight[WORK_NR_COLORS];
 200                                                 /* L: nr of in_flight works */
 201         int                     nr_active;      /* L: nr of active works */
 202         int                     max_active;     /* L: max active works */
 203         struct list_head        delayed_works;  /* L: delayed works */
 204         struct list_head        pwqs_node;      /* WR: node on wq->pwqs */
 205         struct list_head        mayday_node;    /* MD: node on wq->maydays */
 206
 207         /*
 208          * Release of unbound pwq is punted to system_wq.  See put_pwq()
 209          * and pwq_unbound_release_workfn() for details.  pool_workqueue
 210          * itself is also sched-RCU protected so that the first pwq can be
 211          * determined without grabbing wq->mutex.
 212          */
 213         struct work_struct      unbound_release_work;
 214         struct rcu_head         rcu;
 215 } __aligned(1 << WORK_STRUCT_FLAG_BITS);
```
#### `work_struct`

```c
100 struct work_struct {
101         atomic_long_t data;
102         struct list_head entry;
103         work_func_t func;
104 #ifdef CONFIG_LOCKDEP
105         struct lockdep_map lockdep_map;
106 #endif
107 };
```

#### `struct delayed_work`

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

#### Define a delayed worker.

include two parts:

1. timer: to process the 'delay', define a timer with `delayed_work_timer_fn`.
	which just call `__queue_work`.

2. all the other parts are done for `work`.
 
```c
static DECLARE_DELAYED_WORK(dst_gc_work, dst_gc_task);
```

```c
174 #define DECLARE_DELAYED_WORK(n, f)                                      \
175         struct delayed_work n = __DELAYED_WORK_INITIALIZER(n, f, 0)
```

```c
164 #define __DELAYED_WORK_INITIALIZER(n, f, tflags) {                      \
165         .work = __WORK_INITIALIZER((n).work, (f)),                      \
166         .timer = __TIMER_INITIALIZER(delayed_work_timer_fn,             \
167                                      0, (unsigned long)&(n),            \
168                                      (tflags) | TIMER_IRQSAFE),         \
169         }
```

```c
157 #define __WORK_INITIALIZER(n, f) {                                      \
158         .data = WORK_DATA_STATIC_INIT(),                                \
159         .entry  = { &(n).entry, &(n).entry },                           \
160         .func = (f),                                                    \
161         __WORK_INIT_LOCKDEP_MAP(#n, &(n))                               \
162         }
```

```c
109 #define WORK_DATA_INIT()        ATOMIC_LONG_INIT(WORK_STRUCT_NO_POOL)
110 #define WORK_DATA_STATIC_INIT() \
111         ATOMIC_LONG_INIT(WORK_STRUCT_NO_POOL | WORK_STRUCT_STATIC)
```

```c
1444 void delayed_work_timer_fn(unsigned long __data)
1445 {
1446         struct delayed_work *dwork = (struct delayed_work *)__data;
1447
1448         /* should have been called from irqsafe timer with irq already off */
1449         __queue_work(dwork->cpu, dwork->wq, &dwork->work);
1450 }
1451 EXPORT_SYMBOL(delayed_work_timer_fn);
```
#### define a work or deferrable work.
```c
171 #define DECLARE_WORK(n, f)                                              \
172         struct work_struct n = __WORK_INITIALIZER(n, f)
177 #define DECLARE_DEFERRABLE_WORK(n, f)                                   \
178         struct delayed_work n = __DELAYED_WORK_INITIALIZER(n, f, TIMER_DEFERRABLE)
```

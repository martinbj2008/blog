---
layout: post
title: "How Kernel Thread Is Created"
date: 2013-06-27 00:00
comments: true
categories: [sched]
tags: [kernel, schedule, kthread]
---

JUN 27TH, 2013 | COMMENTS
How to create a kernel thread


## create a kthread by `kthread_create`

`kthread_create(threadfn, data, namefmt, arg...)`

example: file drivers/char/apm-emulation.c
```
651 kapmd_tsk = kthread_create(kapmd, NULL, "kapmd");
```
<!-- more -->

## Principle

When kernel bootup, two important threads are created. INIT thread(PID1) and kthreadd(PID2).

PID2 is what we analysis here.

kthread_create_list: all the threads to be created are stored in this list.
To creae a new thread, we just fill the related information in a struct kthread_create_info, and list it to the list, then wait until the thread is ready by mutex.

###kthreadd: PID2 main function

it is a infinite loop, which removes the nodes from kthread_create_list, and creates a thread per node struct kthread_create_info, then notify the kthread is ready.

After the new thread is created by `do_fork`, a common funtion `thread` is call as the new thread entry, not the `threadfn` we give by API `kthread_create`

`thread` will do some common initialization, and notify new thread is ready, and then the real new thread function `threadfn` is called.

All the kernel threads are the children of PID2(kthreadd).

##Data structure

### `kthread_create_lock`
spin lock kthread_create_lock is used to protect the kthread_create_list.

```c
 23 static DEFINE_SPINLOCK(kthread_create_lock);
 24 static LIST_HEAD(kthread_create_list);
 25 struct task_struct *kthreadd_task; <== store PID2.
```

In order to create a kernel thread, this structure MUST be created and filled.

### `kthread_create_info`

```c
 27 struct kthread_create_info
 28 {
 29         /* Information passed to kthread() from kthreadd. */
 30         int (*threadfn)(void *data);
 31         void *data;
 32         int node; <== memory node;
 33
 34         /* Result passed back to kthread_create() from kthreadd. */
 35         struct task_struct *result;
 36         struct completion done;
 37
 38         struct list_head list;
 39 };
```
### `struct kthread`

```c
 41 struct kthread {
 42         unsigned long flags;
 43         unsigned int cpu;
 44         void *data;
 45         struct completion parked;
 46         struct completion exited;
 47 };
```
坑爹: There are a struct kthread and a static int kthread(void *_create). NND，看第一遍的时候没注意到。

## method: kthread_create

```c
 13 #define kthread_create(threadfn, data, namefmt, arg...) \
 14         kthread_create_on_node(threadfn, data, -1, namefmt, ##arg)
```

### Call Trace

```c
> kthread_create
> > kthread_create_on_node
> > > spin_lock(&kthread_create_lock);
> > > list_add_tail(&create.list, &kthread_create_list);
> > > spin_unlock(&kthread_create_lock);
> > > wake_up_process(kthreadd_task); <== hi PID2, 来活了 !!!
> > > wait_for_completion(&create.done); <== waiting for ready.
```

### `kthread_create_on_node`

```
231 /**
232  * kthread_create_on_node - create a kthread.
233  * @threadfn: the function to run until signal_pending(current).
234  * @data: data ptr for @threadfn.
235  * @node: memory node number.
236  * @namefmt: printf-style name for the thread.
237  *
238  * Description: This helper function creates and names a kernel
239  * thread.  The thread will be stopped: use wake_up_process() to start
240  * it.  See also kthread_run().
241  *
242  * If thread is going to be bound on a particular cpu, give its node
243  * in @node, to get NUMA affinity for kthread stack, or else give -1.
244  * When woken, the thread will run @threadfn() with @data as its
245  * argument. @threadfn() can either call do_exit() directly if it is a
246  * standalone thread for which no one will call kthread_stop(), or
247  * return when 'kthread_should_stop()' is true (which means
248  * kthread_stop() has been called).  The return value should be zero
249  * or a negative error number; it will be passed to kthread_stop().
250  *
251  * Returns a task_struct or ERR_PTR(-ENOMEM).
252  */
253 struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
254                                            void *data, int node,
255                                            const char namefmt[],
256                                            ...)
257 {
258         struct kthread_create_info create;
259
260         create.threadfn = threadfn;
261         create.data = data;
262         create.node = node;
263         init_completion(&create.done);
264
265         spin_lock(&kthread_create_lock);
266         list_add_tail(&create.list, &kthread_create_list);
267         spin_unlock(&kthread_create_lock);
268
269         wake_up_process(kthreadd_task);
270         wait_for_completion(&create.done);
271
272         if (!IS_ERR(create.result)) {
273                 static const struct sched_param param = { .sched_priority = 0 };
274                 va_list args;
275
276                 va_start(args, namefmt);
277                 vsnprintf(create.result->comm, sizeof(create.result->comm),
278                           namefmt, args);
279                 va_end(args);
280                 /*
281                  * root may have changed our (kthreadd's) priority or CPU mask.
282                  * The kernel thread should not inherit these properties.
283                  */
284                 sched_setscheduler_nocheck(create.result, SCHED_NORMAL, &param);
285                 set_cpus_allowed_ptr(create.result, cpu_all_mask);
286         }
287         return create.result;
288 }
289 EXPORT_SYMBOL(kthread_create_on_node);
```

### How a kthread is really created.

#### Call Trace

```c
> kthreadd
> > spin_lock(&kthread_create_lock);
> > create = list_entry(kthread_create_list.next, struct kthread_create_info, list);
> > list_del_init(&create->list);
> > spin_unlock(&kthread_create_lock);
> > create_kthread(create);
> > >  kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
> > > > do_fork
> > > > > kthread
> > > > > > __set_current_state(TASK_UNINTERRUPTIBLE);
> > > > > > create->result = current;
> > > > > > complete(&create->done); <=== NOTIFY thread is ready !!!
NOTE: line 224 the pameter kthread in funtion create_thread, is a funtion
```

#### method `kthreadd`
```c
446 int kthreadd(void *unused)
447 {
448         struct task_struct *tsk = current;
449
450         /* Setup a clean context for our children to inherit. */
451         set_task_comm(tsk, "kthreadd");
452         ignore_signals(tsk);
453         set_cpus_allowed_ptr(tsk, cpu_all_mask);
454         set_mems_allowed(node_states[N_MEMORY]);
455
456         current->flags |= PF_NOFREEZE;
457
458         for (;;) {
459                 set_current_state(TASK_INTERRUPTIBLE);
460                 if (list_empty(&kthread_create_list))
461                         schedule();
462                 __set_current_state(TASK_RUNNING);
463
464                 spin_lock(&kthread_create_lock);
465                 while (!list_empty(&kthread_create_list)) {
466                         struct kthread_create_info *create;
467
468                         create = list_entry(kthread_create_list.next,
469                                             struct kthread_create_info, list);
470                         list_del_init(&create->list);
471                         spin_unlock(&kthread_create_lock);
472
473                         create_kthread(create);
474
475                         spin_lock(&kthread_create_lock);
476                 }
477                 spin_unlock(&kthread_create_lock);
478         }
479
480         return 0;
481 }
```
#### `create_kthread`
```c
216 static void create_kthread(struct kthread_create_info *create)
217 {
218         int pid;
219
220 #ifdef CONFIG_NUMA
221         current->pref_node_fork = create->node;
222 #endif
223         /* We want our own signal handler (we take no signals by default). */
224         pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
225         if (pid < 0) {
226                 create->result = ERR_PTR(pid);
227                 complete(&create->done);
228         }
229 }
```

#### `kthread`: do some preparation before really run the threadfn.
```c
175 static int kthread(void *_create)
176 {
177         /* Copy data: it's on kthread's stack */
178         struct kthread_create_info *create = _create;
179         int (*threadfn)(void *data) = create->threadfn;
180         void *data = create->data;
181         struct kthread self;
182         int ret;
183
184         self.flags = 0;
185         self.data = data;
186         init_completion(&self.exited);
187         init_completion(&self.parked);
188         current->vfork_done = &self.exited;
189
190         /* OK, tell user we're spawned, wait for stop or wakeup */
191         __set_current_state(TASK_UNINTERRUPTIBLE);
192         create->result = current;
193         complete(&create->done);   <=== NOTIFY new thread is ready.
194         schedule();
195
196         ret = -EINTR;
197
198         if (!test_bit(KTHREAD_SHOULD_STOP, &self.flags)) {
199                 __kthread_parkme(&self);
200                 ret = threadfn(data);
201         }
202         /* we can't just return, we must preserve "self" on stack */
203         do_exit(ret);
204 }
```

### Where is kthreadd_task created at?

PID2 and `kthreadd_task` is created just after boot up

``` c
364 static noinline void __init_refok rest_init(void)
365 {
366         int pid;
367
368         rcu_scheduler_starting();
369         /*
370          * We need to spawn init first so that it obtains pid 1, however
371          * the init task will end up wanting to create kthreads, which, if
372          * we schedule it before we create kthreadd, will OOPS.
373          */
374         kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
375         numa_default_policy();
376         pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES); <==PID1
377         rcu_read_lock();
378         kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns); <== PID2
379         rcu_read_unlock();
380         complete(&kthreadd_done);
381
382         /*
383          * The boot idle thread must execute schedule()
384          * at least once to get things moving:
385          */
386         init_idle_bootup_task(current);
387         schedule_preempt_disabled();
388         /* Call into cpu_idle with preempt disabled */
389         cpu_startup_entry(CPUHP_ONLINE);
390 }
```

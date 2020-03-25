---
layout: post
title: "worker and worker_thread"
date: 2014-02-18 11:40
comments: true
categories: [IRQs]
tags: [workqueue] 
---

###Summary
The `struct worker` is the really scheudle unit in workqueue.
Each `struct worker` has a corresponding thread(task) by `worker->task`.
A `struct worker` is linked to `struct worker_pool->idle_list` when work is idle.
and moved to `struct worker_pool->busy_hash`.

### `worker_thread` 
1. move `worker` from `pool->idle_list` and clear `worker` 's `WORKER_IDLE` flag.
2. check the `pool` and manage the workers(create/destory)
3. 	Iterate all the `struct work_struct *work` in the `struct worker_pool->worklist`,
	and run them in sequence with `process_one_work(worker, work);`.
4. move `worker` into idle list again.
5. schedule();

<!-- more -->

```c
2270 /**
2271  * worker_thread - the worker thread function
2272  * @__worker: self
2273  *
2274  * The worker thread function.  All workers belong to a worker_pool -
2275  * either a per-cpu one or dynamic unbound one.  These workers process all
2276  * work items regardless of their specific target workqueue.  The only
2277  * exception is work items which belong to workqueues with a rescuer which
2278  * will be explained in rescuer_thread().
2279  *
2280  * Return: 0
2281  */
2282 static int worker_thread(void *__worker)
2283 {
2284         struct worker *worker = __worker;
2285         struct worker_pool *pool = worker->pool;
2286
2287         /* tell the scheduler that this is a workqueue worker */
2288         worker->task->flags |= PF_WQ_WORKER;
2289 woke_up:
2290         spin_lock_irq(&pool->lock);
2291
2292         /* am I supposed to die? */
2293         if (unlikely(worker->flags & WORKER_DIE)) {
2294                 spin_unlock_irq(&pool->lock);
2295                 WARN_ON_ONCE(!list_empty(&worker->entry));
2296                 worker->task->flags &= ~PF_WQ_WORKER;
2297                 return 0;
2298         }
2299
2300         worker_leave_idle(worker);
2301 recheck:
2302         /* no more worker necessary? */
2303         if (!need_more_worker(pool))
2304                 goto sleep;
2305
2306         /* do we need to manage? */
2307         if (unlikely(!may_start_working(pool)) && manage_workers(worker))
2308                 goto recheck;
2309
2310         /*
2311          * ->scheduled list can only be filled while a worker is
2312          * preparing to process a work or actually processing it.
2313          * Make sure nobody diddled with it while I was sleeping.
2314          */
2315         WARN_ON_ONCE(!list_empty(&worker->scheduled));
2316
2317         /*
2318          * Finish PREP stage.  We're guaranteed to have at least one idle
2319          * worker or that someone else has already assumed the manager
2320          * role.  This is where @worker starts participating in concurrency
2321          * management if applicable and concurrency management is restored
2322          * after being rebound.  See rebind_workers() for details.
2323          */
2324         worker_clr_flags(worker, WORKER_PREP | WORKER_REBOUND);
2325
2326         do {
2327                 struct work_struct *work =
2328                         list_first_entry(&pool->worklist,
2329                                          struct work_struct, entry);
2330
2331                 if (likely(!(*work_data_bits(work) & WORK_STRUCT_LINKED))) {
2332                         /* optimization path, not strictly necessary */
2333                         process_one_work(worker, work);
2334                         if (unlikely(!list_empty(&worker->scheduled)))
2335                                 process_scheduled_works(worker);
2336                 } else {
2337                         move_linked_works(work, &worker->scheduled, NULL);
2338                         process_scheduled_works(worker);
2339                 }
2340         } while (keep_working(pool));
2341
2342         worker_set_flags(worker, WORKER_PREP, false);
2343 sleep:
2344         if (unlikely(need_to_manage_workers(pool)) && manage_workers(worker))
2345                 goto recheck;
2346
2347         /*
2348          * pool->lock is held and there's no work to process and no need to
2349          * manage, sleep.  Workers are woken up only while holding
2350          * pool->lock or from local cpu, so setting the current state
2351          * before releasing pool->lock is enough to prevent losing any
2352          * event.
2353          */
2354         worker_enter_idle(worker);
2355         __set_current_state(TASK_INTERRUPTIBLE);
2356         spin_unlock_irq(&pool->lock);
2357         schedule();
2358         goto woke_up;
2359 }
```

### `process_one_work`
```c
2114 /**
2115  * process_one_work - process single work
2116  * @worker: self
2117  * @work: work to process
2118  *
2119  * Process @work.  This function contains all the logics necessary to
2120  * process a single work including synchronization against and
2121  * interaction with other workers on the same cpu, queueing and
2122  * flushing.  As long as context requirement is met, any worker can
2123  * call this function to process a work.
2124  *
2125  * CONTEXT:
2126  * spin_lock_irq(pool->lock) which is released and regrabbed.
2127  */
2128 static void process_one_work(struct worker *worker, struct work_struct *work)
2129 __releases(&pool->lock)
2130 __acquires(&pool->lock)
2131 {
2132         struct pool_workqueue *pwq = get_work_pwq(work);
2133         struct worker_pool *pool = worker->pool;
2134         bool cpu_intensive = pwq->wq->flags & WQ_CPU_INTENSIVE;
2135         int work_color;
2136         struct worker *collision;
2137 #ifdef CONFIG_LOCKDEP
2138         /*
2139          * It is permissible to free the struct work_struct from
2140          * inside the function that is called from it, this we need to
2141          * take into account for lockdep too.  To avoid bogus "held
2142          * lock freed" warnings as well as problems when looking into
2143          * work->lockdep_map, make a copy and use that here.
2144          */
2145         struct lockdep_map lockdep_map;
2146
2147         lockdep_copy_map(&lockdep_map, &work->lockdep_map);
2148 #endif
2149         /*
2150          * Ensure we're on the correct CPU.  DISASSOCIATED test is
2151          * necessary to avoid spurious warnings from rescuers servicing the
2152          * unbound or a disassociated pool.
2153          */
2154         WARN_ON_ONCE(!(worker->flags & WORKER_UNBOUND) &&
2155                      !(pool->flags & POOL_DISASSOCIATED) &&
2156                      raw_smp_processor_id() != pool->cpu);
2157
2158         /*
2159          * A single work shouldn't be executed concurrently by
2160          * multiple workers on a single cpu.  Check whether anyone is
2161          * already processing the work.  If so, defer the work to the
2162          * currently executing one.
2163          */
2164         collision = find_worker_executing_work(pool, work);
2165         if (unlikely(collision)) {
2166                 move_linked_works(work, &collision->scheduled, NULL);
2167                 return;
2168         }
2169
2170         /* claim and dequeue */
2171         debug_work_deactivate(work);
2172         hash_add(pool->busy_hash, &worker->hentry, (unsigned long)work);
2173         worker->current_work = work;
2174         worker->current_func = work->func;
2174         worker->current_func = work->func;
2175         worker->current_pwq = pwq;
2176         work_color = get_work_color(work);
2177
2178         list_del_init(&work->entry);
2179
2180         /*
2181          * CPU intensive works don't participate in concurrency
2182          * management.  They're the scheduler's responsibility.
2183          */
2184         if (unlikely(cpu_intensive))
2185                 worker_set_flags(worker, WORKER_CPU_INTENSIVE, true);
2186
2187         /*
2188          * Unbound pool isn't concurrency managed and work items should be
2189          * executed ASAP.  Wake up another worker if necessary.
2190          */
2191         if ((worker->flags & WORKER_UNBOUND) && need_more_worker(pool))
2192                 wake_up_worker(pool);
2193
2194         /*
2195          * Record the last pool and clear PENDING which should be the last
2196          * update to @work.  Also, do this inside @pool->lock so that
2197          * PENDING and queued state changes happen together while IRQ is
2198          * disabled.
2199          */
2200         set_work_pool_and_clear_pending(work, pool->id);
2201
2202         spin_unlock_irq(&pool->lock);
2203
2204         lock_map_acquire_read(&pwq->wq->lockdep_map);
2205         lock_map_acquire(&lockdep_map);
2206         trace_workqueue_execute_start(work);
2207         worker->current_func(work);
2208         /*
2209          * While we must be careful to not use "work" after this, the trace
2210          * point will only record its address.
2211          */
2212         trace_workqueue_execute_end(work);
2213         lock_map_release(&lockdep_map);
2214         lock_map_release(&pwq->wq->lockdep_map);
2215
2216         if (unlikely(in_atomic() || lockdep_depth(current) > 0)) {
2217                 pr_err("BUG: workqueue leaked lock or atomic: %s/0x%08x/%d\n"
2218                        "     last function: %pf\n",
2219                        current->comm, preempt_count(), task_pid_nr(current),
2220                        worker->current_func);
2221                 debug_show_held_locks(current);
2222                 dump_stack();
2223         }
2224
2225         /*
2226          * The following prevents a kworker from hogging CPU on !PREEMPT
2227          * kernels, where a requeueing work item waiting for something to
2228          * happen could deadlock with stop_machine as such work item could
2229          * indefinitely requeue itself while all other CPUs are trapped in
2230          * stop_machine.
2231          */
2232         cond_resched();
2233
2234         spin_lock_irq(&pool->lock);
2235
2236         /* clear cpu intensive status */
2237         if (unlikely(cpu_intensive))
2238                 worker_clr_flags(worker, WORKER_CPU_INTENSIVE);
2239
2240         /* we're done with it, release */
2241         hash_del(&worker->hentry);
2242         worker->current_work = NULL;
2243         worker->current_func = NULL;
2244         worker->current_pwq = NULL;
2245         worker->desc_valid = false;
2246         pwq_dec_nr_in_flight(pwq, work_color);
2247 }
```

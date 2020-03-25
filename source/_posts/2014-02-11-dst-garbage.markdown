---
layout: post
title: "dst garbage"
date: 2014-02-11 14:22
comments: true
categories: [route]
---

### dst garbage summary
`garbage collection` is a common method used in kernel.
When a object(struct,memeory) become invalid, we need
free them, but the object maybe reference by others.

such as a `dst_entry` is not invalid, and it is still
referenced(used) by others.

then `__dst_free` will be called for this case.
It will first set `dst` to dirty(dead),
and then put it into `dst_garbage.list` by `dst->next`.

Then a workqueue task will check the `dst`'s reference,
and free(destory) it when no reference on it.

Two key struct **`struct dst_garbage`** and **`dst_gc_work`**
<!-- more -->
### `struct dst_garbage`
```c
 38 /*
 39  * We want to keep lock & list close together
 40  * to dirty as few cache lines as possible in __dst_free().
 41  * As this is not a very strong hint, we dont force an alignment on SMP.
 42  */
 43 static struct {
 44         spinlock_t              lock;
 45         struct dst_entry        *list;
 46         unsigned long           timer_inc;
 47         unsigned long           timer_expires;
 48 } dst_garbage = {
 49         .lock = __SPIN_LOCK_UNLOCKED(dst_garbage.lock),
 50         .timer_inc = DST_GC_MAX,
 51 };
```

```c
52 static void dst_gc_task(struct work_struct *work);
 ...
55 static DECLARE_DELAYED_WORK(dst_gc_work, dst_gc_task);
```
```c
217 void __dst_free(struct dst_entry *dst)
218 {
219         spin_lock_bh(&dst_garbage.lock);
220         ___dst_free(dst);
221         dst->next = dst_garbage.list;
222         dst_garbage.list = dst;
223         if (dst_garbage.timer_inc > DST_GC_INC) {
224                 dst_garbage.timer_inc = DST_GC_INC;
225                 dst_garbage.timer_expires = DST_GC_MIN;
226                 mod_delayed_work(system_wq, &dst_gc_work,
227                                  dst_garbage.timer_expires);
228         }
229         spin_unlock_bh(&dst_garbage.lock);
230 }
231 EXPORT_SYMBOL(__dst_free);
```

### `dst_gc_task`

`dst_busy_list` is not initialized?

There are 3 list in this function.
#### `dst_garbage.list`: 
The nodes in this list are added by `__dst_free`.

Eeach round, `dst_gc_task` will check this list.
The dst has no reference will be free(destroy).
the others will be appended to `dst_busy_list`.
So after **a round** , the list will be empty.

#### `dst_busy_list`
After `dst_gc_task` finish, all the referenced `dst` nodes are in this list.

#### `head`
This is temporary list. 
All the referenced `dst` nodes during `dst_gc_task` are in this list.
Before `dst_gc_task` finish, they will be moved to `dst_busy_list`.

#### the main function of workqueue `dst_gc_task`   
1. check the old `dst` nodes,
2. for the un-referenced node, free(destory) it. or 
put them to the temp list `head`.
3. for the node in `dst_garbage.list` do the same 
operation like 2
4. move tmp list `head` to `dst_busy_list`.
5. schedule gc task.

NOTE: BH lock are needed because TX softirq also aceess `dst_garbage`.
```c
 63 static void dst_gc_task(struct work_struct *work)
 64 {
 65         int    delayed = 0;
 66         int    work_performed = 0;
 67         unsigned long expires = ~0L;
 68         struct dst_entry *dst, *next, head;
 69         struct dst_entry *last = &head;
 70
 71         mutex_lock(&dst_gc_mutex);
 72         next = dst_busy_list;
 73
 74 loop:
 75         while ((dst = next) != NULL) {
 76                 next = dst->next;
 77                 prefetch(&next->next);
 78                 cond_resched();
 79                 if (likely(atomic_read(&dst->__refcnt))) {
 80                         last->next = dst;
 81                         last = dst;
 82                         delayed++;
 83                         continue;
 84                 }
 85                 work_performed++;
 86
 87                 dst = dst_destroy(dst);
 88                 if (dst) {
 89                         /* NOHASH and still referenced. Unless it is already
 90                          * on gc list, invalidate it and add to gc list.
 91                          *
 92                          * Note: this is temporary. Actually, NOHASH dst's
 93                          * must be obsoleted when parent is obsoleted.
 94                          * But we do not have state "obsoleted, but
 95                          * referenced by parent", so it is right.
 96                          */
 97                         if (dst->obsolete > 0)
 98                                 continue;
 99
100                         ___dst_free(dst);
101                         dst->next = next;
102                         next = dst;
103                 }
104         }
105
106         spin_lock_bh(&dst_garbage.lock);
107         next = dst_garbage.list;
108         if (next) {
109                 dst_garbage.list = NULL;
110                 spin_unlock_bh(&dst_garbage.lock);
111                 goto loop;
112         }
113         last->next = NULL;
114         dst_busy_list = head.next;
115         if (!dst_busy_list)
116                 dst_garbage.timer_inc = DST_GC_MAX;
117         else {
118                 /*
119                  * if we freed less than 1/10 of delayed entries,
120                  * we can sleep longer.
121                  */
122                 if (work_performed <= delayed/10) {
123                         dst_garbage.timer_expires += dst_garbage.timer_inc;
124                         if (dst_garbage.timer_expires > DST_GC_MAX)
125                                 dst_garbage.timer_expires = DST_GC_MAX;
126                         dst_garbage.timer_inc += DST_GC_INC;
127                 } else {
128                         dst_garbage.timer_inc = DST_GC_INC;
129                         dst_garbage.timer_expires = DST_GC_MIN;
130                 }
131                 expires = dst_garbage.timer_expires;
132                 /*
133                  * if the next desired timer is more than 4 seconds in the
134                  * future then round the timer to whole seconds
135                  */
136                 if (expires > 4*HZ)
137                         expires = round_jiffies_relative(expires);
138                 schedule_delayed_work(&dst_gc_work, expires);
139         }
140
141         spin_unlock_bh(&dst_garbage.lock);
142         mutex_unlock(&dst_gc_mutex);
143 }
```

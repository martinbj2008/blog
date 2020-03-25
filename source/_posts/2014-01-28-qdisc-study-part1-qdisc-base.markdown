---
layout: post
title: "qdisc study part1: qdisc_base"
date: 2014-01-28 11:30
comments: true
categories: [netcore]
---
###`Qdisc_ops` is the core of  a Qdisc.
All kinds of the `Qdisc_ops` are linked in a list by `qdisc_base`.
The **key** item of different `Qdisc_ops` is `id[IFNAMSIZ]`. 

Note: the list is a Singly-linked list, not a common list of kernel.

```c
158 struct Qdisc_ops {
159         struct Qdisc_ops        *next;
160         const struct Qdisc_class_ops    *cl_ops;
161         char                    id[IFNAMSIZ];
162         int                     priv_size;
163
164         int                     (*enqueue)(struct sk_buff *, struct Qdisc *);
165         struct sk_buff *        (*dequeue)(struct Qdisc *);
166         struct sk_buff *        (*peek)(struct Qdisc *);
167         unsigned int            (*drop)(struct Qdisc *);
168
169         int                     (*init)(struct Qdisc *, struct nlattr *arg);
170         void                    (*reset)(struct Qdisc *);
171         void                    (*destroy)(struct Qdisc *);
172         int                     (*change)(struct Qdisc *, struct nlattr *arg);
173         void                    (*attach)(struct Qdisc *);
174
175         int                     (*dump)(struct Qdisc *, struct sk_buff *);
176         int                     (*dump_stats)(struct Qdisc *, struct gnet_dump *);
177
178         struct module           *owner;
179 };
```

### `qdisc_base`
```c
 134 /* The list of all installed queueing disciplines. */
 135
 136 static struct Qdisc_ops *qdisc_base;
```
<!-- more -->

### the default `Qdisc_ops` is `default_qdisc_ops`.
```c
 33 /* Qdisc to use by default */
 34 const struct Qdisc_ops *default_qdisc_ops = &pfifo_fast_ops;
 35 EXPORT_SYMBOL(default_qdisc_ops);
```

```c
526 struct Qdisc_ops pfifo_fast_ops __read_mostly = {
527         .id             =       "pfifo_fast",
528         .priv_size      =       sizeof(struct pfifo_fast_priv),
529         .enqueue        =       pfifo_fast_enqueue,
530         .dequeue        =       pfifo_fast_dequeue,
531         .peek           =       pfifo_fast_peek,
532         .init           =       pfifo_fast_init,
533         .reset          =       pfifo_fast_reset,
534         .dump           =       pfifo_fast_dump,
535         .owner          =       THIS_MODULE,
536 };
```


### `register_qdisc`
The list is not a sorted list, it would be better with sorted list,
but these reg/unreg functions are rarely used.
do some basic check for the methods of `Qdisc_ops`,
and set some default values.

for example, method `peek` must be empty if method `dequeue` is empty.

```c
 138 /* Register/uregister queueing discipline */
 139
 140 int register_qdisc(struct Qdisc_ops *qops)
 141 {
 142         struct Qdisc_ops *q, **qp;
 143         int rc = -EEXIST;
 144
 145         write_lock(&qdisc_mod_lock);
 146         for (qp = &qdisc_base; (q = *qp) != NULL; qp = &q->next)
 147                 if (!strcmp(qops->id, q->id))
 148                         goto out;
 149
 150         if (qops->enqueue == NULL)
 151                 qops->enqueue = noop_qdisc_ops.enqueue;
 152         if (qops->peek == NULL) {
 153                 if (qops->dequeue == NULL)
 154                         qops->peek = noop_qdisc_ops.peek;
 155                 else
 156                         goto out_einval;
 157         }
 158         if (qops->dequeue == NULL)
 159                 qops->dequeue = noop_qdisc_ops.dequeue;
 160
 161         if (qops->cl_ops) {
 162                 const struct Qdisc_class_ops *cops = qops->cl_ops;
 163
 164                 if (!(cops->get && cops->put && cops->walk && cops->leaf))
 165                         goto out_einval;
 166
 167                 if (cops->tcf_chain && !(cops->bind_tcf && cops->unbind_tcf))
 168                         goto out_einval;
 169         }
 170
 171         qops->next = NULL;
 172         *qp = qops;
 173         rc = 0;
 174 out:
 175         write_unlock(&qdisc_mod_lock);
 176         return rc;
 177
 178 out_einval:
 179         rc = -EINVAL;
 180         goto out;
 181 }
 182 EXPORT_SYMBOL(register_qdisc);
```

### `unregister_qdisc`
```c
 184 int unregister_qdisc(struct Qdisc_ops *qops)
 185 {
 186         struct Qdisc_ops *q, **qp;
 187         int err = -ENOENT;
 188
 189         write_lock(&qdisc_mod_lock);
 190         for (qp = &qdisc_base; (q = *qp) != NULL; qp = &q->next)
 191                 if (q == qops)
 192                         break;
 193         if (q) {
 194                 *qp = q->next;
 195                 q->next = NULL;
 196                 err = 0;
 197         }
 198         write_unlock(&qdisc_mod_lock);
 199         return err;
 200 }
 201 EXPORT_SYMBOL(unregister_qdisc);
```

### all `Qdisc_ops` 
```c
junwei@localhost:~/git/linux$ grep register_qdisc net/ -Rw
net/sched/sch_sfb.c:    return register_qdisc(&sfb_qdisc_ops);
net/sched/sch_fq.c:     ret = register_qdisc(&fq_qdisc_ops);
net/sched/sch_choke.c:  return register_qdisc(&choke_qdisc_ops);
net/sched/sch_codel.c:  return register_qdisc(&codel_qdisc_ops);
net/sched/sch_qfq.c:    return register_qdisc(&qfq_qdisc_ops);
net/sched/sch_atm.c:    return register_qdisc(&atm_qdisc_ops);
net/sched/sch_multiq.c: return register_qdisc(&multiq_qdisc_ops);
net/sched/sch_cbq.c:    return register_qdisc(&cbq_qdisc_ops);
net/sched/sch_fq_codel.c:       return register_qdisc(&fq_codel_qdisc_ops);
net/sched/sch_dsmark.c: return register_qdisc(&dsmark_qdisc_ops);
net/sched/sch_hfsc.c:   return register_qdisc(&hfsc_qdisc_ops);
net/sched/sch_htb.c:    return register_qdisc(&htb_qdisc_ops);
net/sched/sch_api.c:int register_qdisc(struct Qdisc_ops *qops)
net/sched/sch_api.c:EXPORT_SYMBOL(register_qdisc);
net/sched/sch_api.c:    register_qdisc(&pfifo_fast_ops);
net/sched/sch_api.c:    register_qdisc(&pfifo_qdisc_ops);
net/sched/sch_api.c:    register_qdisc(&bfifo_qdisc_ops);
net/sched/sch_api.c:    register_qdisc(&pfifo_head_drop_qdisc_ops);
net/sched/sch_api.c:    register_qdisc(&mq_qdisc_ops);
net/sched/sch_blackhole.c:      return register_qdisc(&blackhole_qdisc_ops);
net/sched/sch_drr.c:    return register_qdisc(&drr_qdisc_ops);
net/sched/sch_netem.c:  return register_qdisc(&netem_qdisc_ops);
net/sched/sch_prio.c:   return register_qdisc(&prio_qdisc_ops);
net/sched/sch_gred.c:   return register_qdisc(&gred_qdisc_ops);
net/sched/sch_ingress.c:        return register_qdisc(&ingress_qdisc_ops);
net/sched/sch_mqprio.c: return register_qdisc(&mqprio_qdisc_ops);
net/sched/sch_sfq.c:    return register_qdisc(&sfq_qdisc_ops);
net/sched/sch_tbf.c:    return register_qdisc(&tbf_qdisc_ops);
net/sched/sch_red.c:    return register_qdisc(&red_qdisc_ops);
net/sched/sch_plug.c:   return register_qdisc(&plug_qdisc_ops);
net/sched/sch_teql.c:           err = register_qdisc(&master->qops);
junwei@localhost:~/git/linux$
```

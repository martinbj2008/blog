---
layout: post
title: "How to select tcp client port"
date: 2015-09-09 16:42:00 +0800
comments: true
categories: [socket]
tags: [socket]
---

upstream source v4.2+(commit ID:a794b4f).
```

#### key var
##### tcp_death_row
```
  97 struct inet_hashinfo tcp_hashinfo;
```
#### callback
```
> connect syscall
> > sock->ops->connect
==> tcp_v4_connect
> > > inet_hash_connect
> > > > port_offset = inet_sk_port_offset(sk);
> > > > __inet_hash_connect
> > > > > inet_get_local_port_range
> > > > > for each port in range: start with port_offset.
> > > > > > inet_is_local_reserved_port //skip the reserved ports.
> > > > > >

There is a important data struct `struct inet_hashinfo`
其对应的变量是 `tcp_hashinfo`.

共包含3部分:

1. ehash: establish hash. netnamspace，saddr, daddr, sport, dport. 每个struct sock, 通过&sk->sk_nulls_node，挂接到hash链上.
2. bhash: 使用netnamespace和localport作为key。
	hashlist节点是struct inet_bind_bucket，通过其node域链接到hash链上。
	一个节点对应一个local port。 节点下又一个owner 链。num_owners表示owner链的长度。
	每个struct sock,通过sk->sk_bind_node，链接到owner链上。
3. listening_hash: 待后续完善。tcp server使用。

<!-- more -->

### related source
#### data struct 

```
 34 struct inet_timewait_death_row {
 35         atomic_t                tw_count;
 36
 37         struct inet_hashinfo    *hashinfo ____cacheline_aligned_in_smp;
 38         int                     sysctl_tw_recycle;
 39         int                     sysctl_max_tw_buckets;
 40 };
```

```
116 struct inet_hashinfo {
117         /* This is for sockets with full identity only.  Sockets here will
118          * always be without wildcards and will have the following invariant:
119          *
120          *          TCP_ESTABLISHED <= sk->sk_state < TCP_CLOSE
121          *
122          */
123         struct inet_ehash_bucket        *ehash;
124         spinlock_t                      *ehash_locks;
125         unsigned int                    ehash_mask;
126         unsigned int                    ehash_locks_mask;
127
128         /* Ok, let's try this, I give up, we do need a local binding
129          * TCP hash as well as the others for fast bind/connect.
130          */
131         struct inet_bind_hashbucket     *bhash;
132
133         unsigned int                    bhash_size;
134         /* 4 bytes hole on 64 bit */
135
136         struct kmem_cache               *bind_bucket_cachep;
137
138         /* All the above members are written once at bootup and
139          * never written again _or_ are predominantly read-access.
140          *
141          * Now align to a new cache line as all the following members
142          * might be often dirty.
143          */
144         /* All sockets in TCP_LISTEN state will be in here.  This is the only
145          * table where wildcard'd TCP sockets can exist.  Hash function here
146          * is just local port number.
147          */
148         struct inet_listen_hashbucket   listening_hash[INET_LHTABLE_SIZE]
149                                         ____cacheline_aligned_in_smp;
150 };
```

##### system call connect
```
SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
                int, addrlen)
{
 ...
        err = sock->ops->connect(sock, (struct sockaddr *)&address, addrlen,
                                 sock->file->f_flags);
...
}
```

#### 
```
 140 /* This will initiate an outgoing connection. */
 141 int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
 142 {
...
 215         /* Socket identity is still unknown (sport may be zero).
 216          * However we set state to SYN-SENT and not releasing socket
 217          * lock select source port, enter ourselves into the hash tables and
 218          * complete initialization after this.
 219          */
 220         tcp_set_state(sk, TCP_SYN_SENT);
 221         err = inet_hash_connect(&tcp_death_row, sk);
 222         if (err)
 223                 goto failure;
...
 264 }
 265 EXPORT_SYMBOL(tcp_v4_connect);
```

#####
```
584 /*
585  * Bind a port for a connect operation and hash it.
586  */
587 int inet_hash_connect(struct inet_timewait_death_row *death_row,
588                       struct sock *sk)
589 {
590         u32 port_offset = 0;
591
592         if (!inet_sk(sk)->inet_num)
593                 port_offset = inet_sk_port_offset(sk);
594         return __inet_hash_connect(death_row, sk, port_offset,
595                                    __inet_check_established);
596 }
597 EXPORT_SYMBOL_GPL(inet_hash_connect);
```


##### `__inet_hash_connect`
```
a. 为了安全性，选区一个随机的端口偏移。
b. 首先根据inet_get_local_port_range获得本机tcp端口的合法范围。
c. 从随机偏移的端口开始，便利所有的port
	for each port in ragne 
	c.1 判断端口是否为保留的端口。 inet_is_local_reserved_port
		c.1.1 如果是保留节点，下一个节点
	c.2 	根据netnamespace和port计算hash值，并去对应的bhash桶的头。
	c.3 	便利bhash链上对应的每个节点。
		c.3.1 比较每个节点的(port和netnamespace) vs 当前的port和netnamespace
		c.3.2 如果不相等，下一个节点，否则检查是否为check_established
		c.3.2 否则检查是否为check_established
			c.3.2.1 如果检查失败，下一个节点
			c.3.2.2 ture goto OK
	c.4	遍历玩bhash节点，未发现该netnamespace下的port被占用。
		调用inet_bind_bucket_create，占用该netnamespace下的port
		goto OK;
d. 遍历完成，没有合适的节点。 return -EADDRNOTAVAIL;

OK：
	找到合适的port。更新ehash和bhash
	inet_bind_hash(sk, tb, port);
```


```
476 int __inet_hash_connect(struct inet_timewait_death_row *death_row,
477                 struct sock *sk, u32 port_offset,
478                 int (*check_established)(struct inet_timewait_death_row *,
479                         struct sock *, __u16, struct inet_timewait_sock **))
480 {
481         struct inet_hashinfo *hinfo = death_row->hashinfo;
482         const unsigned short snum = inet_sk(sk)->inet_num;
483         struct inet_bind_hashbucket *head;
484         struct inet_bind_bucket *tb;
485         int ret;
486         struct net *net = sock_net(sk);
487
488         if (!snum) {
489                 int i, remaining, low, high, port;
490                 static u32 hint;
491                 u32 offset = hint + port_offset;
492                 struct inet_timewait_sock *tw = NULL;
493
494                 inet_get_local_port_range(net, &low, &high);
495                 remaining = (high - low) + 1;
496
497                 /* By starting with offset being an even number,
498                  * we tend to leave about 50% of ports for other uses,
499                  * like bind(0).
500                  */
501                 offset &= ~1;
502
503                 local_bh_disable();
504                 for (i = 0; i < remaining; i++) {
505                         port = low + (i + offset) % remaining;
506                         if (inet_is_local_reserved_port(net, port))
507                                 continue;
508                         head = &hinfo->bhash[inet_bhashfn(net, port,
509                                         hinfo->bhash_size)];
510                         spin_lock(&head->lock);
511
512                         /* Does not bother with rcv_saddr checks,
513                          * because the established check is already
514                          * unique enough.
515                          */
516                         inet_bind_bucket_for_each(tb, &head->chain) {
517                                 if (net_eq(ib_net(tb), net) &&
518                                     tb->port == port) {
519                                         if (tb->fastreuse >= 0 ||
520                                             tb->fastreuseport >= 0)
521                                                 goto next_port;
522                                         WARN_ON(hlist_empty(&tb->owners));
523                                         if (!check_established(death_row, sk,
524                                                                 port, &tw))
525                                                 goto ok;
526                                         goto next_port;
527                                 }
528                         }
529
530                         tb = inet_bind_bucket_create(hinfo->bind_bucket_cachep,
531                                         net, head, port);
532                         if (!tb) {
533                                 spin_unlock(&head->lock);
534                                 break;
535                         }
536                         tb->fastreuse = -1;
537                         tb->fastreuseport = -1;
538                         goto ok;
539
540                 next_port:
541                         spin_unlock(&head->lock);
542                 }
543                 local_bh_enable();
544
545                 return -EADDRNOTAVAIL;
546
547 ok:
548                 hint += (i + 2) & ~1;
549
550                 /* Head lock still held and bh's disabled */
551                 inet_bind_hash(sk, tb, port);
552                 if (sk_unhashed(sk)) {
553                         inet_sk(sk)->inet_sport = htons(port);
554                         __inet_hash_nolisten(sk, (struct sock *)tw);
555                 }
556                 if (tw)
557                         inet_twsk_bind_unhash(tw, hinfo);
558                 spin_unlock(&head->lock);
559
560                 if (tw)
561                         inet_twsk_deschedule_put(tw);
562
563                 ret = 0;
564                 goto out;
565         }
566
567         head = &hinfo->bhash[inet_bhashfn(net, snum, hinfo->bhash_size)];
568         tb  = inet_csk(sk)->icsk_bind_hash;
569         spin_lock_bh(&head->lock);
570         if (sk_head(&tb->owners) == sk && !sk->sk_bind_node.next) {
571                 __inet_hash_nolisten(sk, NULL);
572                 spin_unlock_bh(&head->lock);
573                 return 0;
574         } else {
575                 spin_unlock(&head->lock);
576                 /* No definite answer... Walk to established hash table */
577                 ret = check_established(death_row, sk, snum, NULL);
578 out:
579                 local_bh_enable();
580                 return ret;
581         }
582 }
```


```
326 /* called with local bh disabled */
327 static int __inet_check_established(struct inet_timewait_death_row *death_row,
328                                     struct sock *sk, __u16 lport,
329                                     struct inet_timewait_sock **twp)
330 {
331         struct inet_hashinfo *hinfo = death_row->hashinfo;
332         struct inet_sock *inet = inet_sk(sk);
333         __be32 daddr = inet->inet_rcv_saddr;
334         __be32 saddr = inet->inet_daddr;
335         int dif = sk->sk_bound_dev_if;
336         INET_ADDR_COOKIE(acookie, saddr, daddr);
337         const __portpair ports = INET_COMBINED_PORTS(inet->inet_dport, lport);
338         struct net *net = sock_net(sk);
339         unsigned int hash = inet_ehashfn(net, daddr, lport,
340                                          saddr, inet->inet_dport);
341         struct inet_ehash_bucket *head = inet_ehash_bucket(hinfo, hash);
342         spinlock_t *lock = inet_ehash_lockp(hinfo, hash);
343         struct sock *sk2;
344         const struct hlist_nulls_node *node;
345         struct inet_timewait_sock *tw = NULL;
346
347         spin_lock(lock);
348
349         sk_nulls_for_each(sk2, node, &head->chain) {
350                 if (sk2->sk_hash != hash)
351                         continue;
352
353                 if (likely(INET_MATCH(sk2, net, acookie,
354                                          saddr, daddr, ports, dif))) {
355                         if (sk2->sk_state == TCP_TIME_WAIT) {
356                                 tw = inet_twsk(sk2);
357                                 if (twsk_unique(sk, sk2, twp))
358                                         break;
359                         }
360                         goto not_unique;
361                 }
362         }
363
364         /* Must record num and sport now. Otherwise we will see
365          * in hash table socket with a funny identity.
366          */
367         inet->inet_num = lport;
368         inet->inet_sport = htons(lport);
369         sk->sk_hash = hash;
370         WARN_ON(!sk_unhashed(sk));
371         __sk_nulls_add_node_rcu(sk, &head->chain);
372         if (tw) {
373                 sk_nulls_del_node_init_rcu((struct sock *)tw);
374                 NET_INC_STATS_BH(net, LINUX_MIB_TIMEWAITRECYCLED);
375         }
376         spin_unlock(lock);
377         sock_prot_inuse_add(sock_net(sk), sk->sk_prot, 1);
378
379         if (twp) {
380                 *twp = tw;
381         } else if (tw) {
382                 /* Silly. Should hash-dance instead... */
383                 inet_twsk_deschedule_put(tw);
384         }
385         return 0;
386
387 not_unique:
388         spin_unlock(lock);
389         return -EADDRNOTAVAIL;
390 }
```

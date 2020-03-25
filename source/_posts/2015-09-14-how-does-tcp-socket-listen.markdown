---
layout: post
title: "how does tcp socket listen"
date: 2015-09-14 14:41:02 +0800
comments: true
categories: [socket]
---

调用inet_hash将对应的socket挂到一个chainlist上
### Call treace
```
 inet_listen
  inet_csk_listen_start
    sk->sk_prot->hash(sk); 相当于inet_hash
```

<!-- more -->

#####
```
 194 int inet_listen(struct socket *sock, int backlog)
 195 {
 ...
 213         if (old_state != TCP_LISTEN) {
...
 214                err = inet_csk_listen_start(sk, backlog);
 237                 if (err)
 238                         goto out;
 239         }
 ...
 245         return err;
 246 }
 247 EXPORT_SYMBOL(inet_listen);
```
##### `inet_csk_listen_start`
```
794 int inet_csk_listen_start(struct sock *sk, const int nr_table_entries)
795 {
796         struct inet_sock *inet = inet_sk(sk);
797         struct inet_connection_sock *icsk = inet_csk(sk);
798         int rc = reqsk_queue_alloc(&icsk->icsk_accept_queue, nr_table_entries);
799
800         if (rc != 0)
801                 return rc;
802
803         sk->sk_max_ack_backlog = 0;
804         sk->sk_ack_backlog = 0;
805         inet_csk_delack_init(sk);
806
807         /* There is race window here: we announce ourselves listening,
808          * but this transition is still not validated by get_port().
809          * It is OK, because this socket enters to hash table only
810          * after validation is complete.
811          */
812         sk->sk_state = TCP_LISTEN;
813         if (!sk->sk_prot->get_port(sk, inet->inet_num)) {
814                 inet->inet_sport = htons(inet->inet_num);
815
816                 sk_dst_reset(sk);
817                 sk->sk_prot->hash(sk);
818
819                 return 0;
820         }
821
822         sk->sk_state = TCP_CLOSE;
823         __reqsk_queue_destroy(&icsk->icsk_accept_queue);
824         return -EADDRINUSE;
825 }
826 EXPORT_SYMBOL_GPL(inet_csk_listen_start);
```

```
426 void __inet_hash(struct sock *sk, struct sock *osk)
427 {
428         struct inet_hashinfo *hashinfo = sk->sk_prot->h.hashinfo;
429         struct inet_listen_hashbucket *ilb;
430
431         if (sk->sk_state != TCP_LISTEN)
432                 return __inet_hash_nolisten(sk, osk);
433
434         WARN_ON(!sk_unhashed(sk));
435         ilb = &hashinfo->listening_hash[inet_sk_listen_hashfn(sk)];
436
437         spin_lock(&ilb->lock);
438         __sk_nulls_add_node_rcu(sk, &ilb->head);
439         sock_prot_inuse_add(sock_net(sk), sk->sk_prot, 1);
440         spin_unlock(&ilb->lock);
441 }
442 EXPORT_SYMBOL(__inet_hash);
443
444 void inet_hash(struct sock *sk)
445 {
446         if (sk->sk_state != TCP_CLOSE) {
447                 local_bh_disable();
448                 __inet_hash(sk, NULL);
449                 local_bh_enable();
450         }
451 }
452 EXPORT_SYMBOL_GPL(inet_hash);
```

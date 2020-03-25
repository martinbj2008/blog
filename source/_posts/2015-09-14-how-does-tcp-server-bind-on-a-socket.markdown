---
layout: post
title: "how does tcp server bind on a socket"
date: 2015-09-14 10:10:13 +0800
comments: true
categories: [socket]
---

bind的port是通过`tcp_hashinfo`里的bhash管理的。   
跟tcp client的端口号管理有一样。

注：bind 系统调用时，还不会把当前的socket挂载到listen队列上，需要等待listen系统调用。
bind系统调用只是把对应的这个端口給占用上了， 其他程序没发bind。
<!-- more -->

##### call trace
case1: 如果是第一次bind，并且没有client占用这个port。   
在[inet_csk_get_port]()遍历bhash里对应的hash链表，   
因为首次bind，所以tb没有找到。   
[inet_bind_bucket_create]创建tb并挂载到bhash的链表里。 
[inet_bind_hash]将socket挂载到tb对应的ownerlist上。
```
   system call bind
     inet_bind
       sk->sk_prot->get_port 相当于 inet_csk_get_port
         inet_bind_bucket_create
         inet_bind_hash
```

case 2: bind+reuseport。close socket后再次bind一个新的soket到同一个port
通过`inet_csk_bind_conflict`保证port之间冲突的检查。
```
   system call bind
     inet_bind
       sk->sk_prot->get_port 相当于 inet_csk_get_port
         inet_csk(sk)->icsk_af_ops->bind_conflict 相当于inet_csk_bind_conflict
```

case 3: 如果bind的port，之前已经有client使用了。
那么对应tb的`fastreuse`应该是-1,所以可以重用。bind之后会被设置为0.


####
```
1361 /*
1362  *      Bind a name to a socket. Nothing much to do here since it's
1363  *      the protocol's responsibility to handle the local address.
1364  *
1365  *      We move the socket address to kernel space before we call
1366  *      the protocol layer (having also checked the address is ok).
1367  */
1368
1369 SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
1370 {
...
1383                                 err = sock->ops->bind(sock,
1384                                                       (struct sockaddr *)
1385                                                       &address, addrlen);
...
1390 }
```
####

```
 901 const struct proto_ops inet_stream_ops = {
 902         .family            = PF_INET,
 903         .owner             = THIS_MODULE,
 904         .release           = inet_release,
 905         .bind              = inet_bind,
```

<span id="inetBind">
```
 423 int inet_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
 424 {
 425         struct sockaddr_in *addr = (struct sockaddr_in *)uaddr;
 426         struct sock *sk = sock->sk;
 427         struct inet_sock *inet = inet_sk(sk);
 428         struct net *net = sock_net(sk);
 429         unsigned short snum;
 430         int chk_addr_ret;
 431         u32 tb_id = RT_TABLE_LOCAL;
 432         int err;
 433
 434         /* If the socket has its own bind function then use it. (RAW) */
 435         if (sk->sk_prot->bind) {
 436                 err = sk->sk_prot->bind(sk, uaddr, addr_len);
 437                 goto out;
 438         }
 439         err = -EINVAL;
 440         if (addr_len < sizeof(struct sockaddr_in))
 441                 goto out;
 442
 443         if (addr->sin_family != AF_INET) {
 444                 /* Compatibility games : accept AF_UNSPEC (mapped to AF_INET)
 445                  * only if s_addr is INADDR_ANY.
 446                  */
 447                 err = -EAFNOSUPPORT;
 448                 if (addr->sin_family != AF_UNSPEC ||
 449                     addr->sin_addr.s_addr != htonl(INADDR_ANY))
 450                         goto out;
 451         }
 452
 453         tb_id = vrf_dev_table_ifindex(net, sk->sk_bound_dev_if) ? : tb_id;
 454         chk_addr_ret = inet_addr_type_table(net, addr->sin_addr.s_addr, tb_id);
 455
 456         /* Not specified by any standard per-se, however it breaks too
 457          * many applications when removed.  It is unfortunate since
 458          * allowing applications to make a non-local bind solves
 459          * several problems with systems using dynamic addressing.
 460          * (ie. your servers still start up even if your ISDN link
 461          *  is temporarily down)
 462          */
 463         err = -EADDRNOTAVAIL;
 464         if (!net->ipv4.sysctl_ip_nonlocal_bind &&
 465             !(inet->freebind || inet->transparent) &&
 466             addr->sin_addr.s_addr != htonl(INADDR_ANY) &&
 467             chk_addr_ret != RTN_LOCAL &&
 468             chk_addr_ret != RTN_MULTICAST &&
 469             chk_addr_ret != RTN_BROADCAST)
 470                 goto out;
 471
 472         snum = ntohs(addr->sin_port);
 473         err = -EACCES;
 474         if (snum && snum < PROT_SOCK &&
 475             !ns_capable(net->user_ns, CAP_NET_BIND_SERVICE))
 476                 goto out;
 477
 478         /*      We keep a pair of addresses. rcv_saddr is the one
 479          *      used by hash lookups, and saddr is used for transmit.
 480          *
 481          *      In the BSD API these are the same except where it
 482          *      would be illegal to use them (multicast/broadcast) in
 483          *      which case the sending device address is used.
 484          */
 485         lock_sock(sk);
 486
 487         /* Check these errors (active socket, double bind). */
 488         err = -EINVAL;
 489         if (sk->sk_state != TCP_CLOSE || inet->inet_num)
 490                 goto out_release_sock;
 491
 492         inet->inet_rcv_saddr = inet->inet_saddr = addr->sin_addr.s_addr;
 493         if (chk_addr_ret == RTN_MULTICAST || chk_addr_ret == RTN_BROADCAST)
 494                 inet->inet_saddr = 0;  /* Use device */
 495
 496         /* Make sure we are allowed to bind here. */
 497         if ((snum || !inet->bind_address_no_port) &&
 498             sk->sk_prot->get_port(sk, snum)) {
 499                 inet->inet_saddr = inet->inet_rcv_saddr = 0;
 500                 err = -EADDRINUSE;
 501                 goto out_release_sock;
 502         }
 503
 504         if (inet->inet_rcv_saddr)
 505                 sk->sk_userlocks |= SOCK_BINDADDR_LOCK;
 506         if (snum)
 507                 sk->sk_userlocks |= SOCK_BINDPORT_LOCK;
 508         inet->inet_sport = htons(inet->inet_num);
 509         inet->inet_daddr = 0;
 510         inet->inet_dport = 0;
 511         sk_dst_reset(sk);
 512         err = 0;
 513 out_release_sock:
 514         release_sock(sk);
 515 out:
 516         return err;
 517 }
 518 EXPORT_SYMBOL(inet_bind);
```
</span>

#### ``inet_csk_get_port`
<span id="inetCSKgetPort">
```
 90 /* Obtain a reference to a local port for the given sock,
 91  * if snum is zero it means select any available local port.
 92  */
 93 int inet_csk_get_port(struct sock *sk, unsigned short snum)
 94 {
 95         struct inet_hashinfo *hashinfo = sk->sk_prot->h.hashinfo;
 96         struct inet_bind_hashbucket *head;
 97         struct inet_bind_bucket *tb;
 98         int ret, attempts = 5;
 99         struct net *net = sock_net(sk);
100         int smallest_size = -1, smallest_rover;
101         kuid_t uid = sock_i_uid(sk);
102         int attempt_half = (sk->sk_reuse == SK_CAN_REUSE) ? 1 : 0;
103
104         local_bh_disable();
105         if (!snum) { 
...     //bind传入固定的port值，snum不会为零，所以忽略自动分配port的情况，
177         } else {
178 have_snum:
179                 head = &hashinfo->bhash[inet_bhashfn(net, snum,
180                                 hashinfo->bhash_size)];
181                 spin_lock(&head->lock);
182                 inet_bind_bucket_for_each(tb, &head->chain)
183                         if (net_eq(ib_net(tb), net) && tb->port == snum)
184                                 goto tb_found;
185         }
186         tb = NULL;
187         goto tb_not_found;
188 tb_found:
189         if (!hlist_empty(&tb->owners)) {
190                 if (sk->sk_reuse == SK_FORCE_REUSE)
191                         goto success;
192
193                 if (((tb->fastreuse > 0 &&
194                       sk->sk_reuse && sk->sk_state != TCP_LISTEN) ||
195                      (tb->fastreuseport > 0 &&
196                       sk->sk_reuseport && uid_eq(tb->fastuid, uid))) &&
197                     smallest_size == -1) {
198                         goto success;
199                 } else {
200                         ret = 1;
201                         if (inet_csk(sk)->icsk_af_ops->bind_conflict(sk, tb, true)) {
202                                 if (((sk->sk_reuse && sk->sk_state != TCP_LISTEN) ||
203                                      (tb->fastreuseport > 0 &&
204                                       sk->sk_reuseport && uid_eq(tb->fastuid, uid))) &&
205                                     smallest_size != -1 && --attempts >= 0) {
206                                         spin_unlock(&head->lock);
207                                         goto again;
208                                 }
209
210                                 goto fail_unlock;
211                         }
212                 }
213         }
214 tb_not_found:
215         ret = 1;
216         if (!tb && (tb = inet_bind_bucket_create(hashinfo->bind_bucket_cachep,
217                                         net, head, snum)) == NULL)
218                 goto fail_unlock;
219         if (hlist_empty(&tb->owners)) {
220                 if (sk->sk_reuse && sk->sk_state != TCP_LISTEN)
221                         tb->fastreuse = 1;
222                 else
223                         tb->fastreuse = 0;
224                 if (sk->sk_reuseport) {
225                         tb->fastreuseport = 1;
226                         tb->fastuid = uid;
227                 } else
228                         tb->fastreuseport = 0;
229         } else {
230                 if (tb->fastreuse &&
231                     (!sk->sk_reuse || sk->sk_state == TCP_LISTEN))
232                         tb->fastreuse = 0;
233                 if (tb->fastreuseport &&
234                     (!sk->sk_reuseport || !uid_eq(tb->fastuid, uid)))
235                         tb->fastreuseport = 0;
236         }
237 success:
238         if (!inet_csk(sk)->icsk_bind_hash)
239                 inet_bind_hash(sk, tb, snum);
240         WARN_ON(inet_csk(sk)->icsk_bind_hash != tb);
241         ret = 0;
242
243 fail_unlock:
244         spin_unlock(&head->lock);
245 fail:
246         local_bh_enable();
247         return ret;
248 }
249 EXPORT_SYMBOL_GPL(inet_csk_get_port);
```
</span>

#### `inet_bind_bucket_create`
<span id="inetBindBucketCreate">
```
 57 /*
 58  * Allocate and initialize a new local port bind bucket.
 59  * The bindhash mutex for snum's hash chain must be held here.
 60  */
 61 struct inet_bind_bucket *inet_bind_bucket_create(struct kmem_cache *cachep,
 62                                                  struct net *net,
 63                                                  struct inet_bind_hashbucket *head,
 64                                                  const unsigned short snum)
 65 {
 66         struct inet_bind_bucket *tb = kmem_cache_alloc(cachep, GFP_ATOMIC);
 67
 68         if (tb) {
 69                 write_pnet(&tb->ib_net, net);
 70                 tb->port      = snum;
 71                 tb->fastreuse = 0;
 72                 tb->fastreuseport = 0;
 73                 tb->num_owners = 0;
 74                 INIT_HLIST_HEAD(&tb->owners);
 75                 hlist_add_head(&tb->node, &head->chain);
 76         }
 77         return tb;
 78 }
```
</span>


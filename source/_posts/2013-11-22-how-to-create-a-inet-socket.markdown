---
layout: post
title: "how to create a inet socket"
date: 2013-11-22 11:32
comments: true
categories: [socket]
tags: [inet socket, socket]
---

###  Data Structure
``` c
1019 static const struct net_proto_family inet_family_ops = {
1020         .family = PF_INET,
1021         .create = inet_create,
1022         .owner  = THIS_MODULE,
1023 };
```

### call trace
```c
> inet_create
> > search the right inetsw array element with type/protocol
> > sock->ops = answer->ops;
> > k = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot);
> > sock_init_data(sock, sk);
> > init the sk
```
<!-- more -->

### Initialization of `inet_family_ops`
```c
> static int __init inet_init(void)
> > (void)sock_register(&inet_family_ops);
```

### `inet_create`
```c
 271 /*
 272  *      Create an inet socket.
 273  */
 274 
 275 static int inet_create(struct net *net, struct socket *sock, int protocol,
 276                        int kern)
 277 {
 278         struct sock *sk;
 279         struct inet_protosw *answer;
 280         struct inet_sock *inet;
 281         struct proto *answer_prot;
 282         unsigned char answer_flags;
 283         char answer_no_check;
 284         int try_loading_module = 0;
 285         int err;
 286 
 287         if (unlikely(!inet_ehash_secret))
 288                 if (sock->type != SOCK_RAW && sock->type != SOCK_DGRAM)
 289                         build_ehash_secret();
 290 
 291         sock->state = SS_UNCONNECTED;
 292 
 293         /* Look for the requested type/protocol pair. */
 294 lookup_protocol:
 295         err = -ESOCKTNOSUPPORT;
 296         rcu_read_lock();
 297         list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {
 298 
 299                 err = 0;
 300                 /* Check the non-wild match. */
 301                 if (protocol == answer->protocol) {
 302                         if (protocol != IPPROTO_IP)
 303                                 break;
 304                 } else {
 305                         /* Check for the two wild cases. */
 306                         if (IPPROTO_IP == protocol) {
 307                                 protocol = answer->protocol;
 308                                 break;
 309                         }
 310                         if (IPPROTO_IP == answer->protocol)
 311                                 break;
 312                 }
 313                 err = -EPROTONOSUPPORT;
 314         }
 315 
 316         if (unlikely(err)) {
 317                 if (try_loading_module < 2) {
 318                         rcu_read_unlock();
 319                         /*
 320                          * Be more specific, e.g. net-pf-2-proto-132-type-1
 321                          * (net-pf-PF_INET-proto-IPPROTO_SCTP-type-SOCK_STREAM)
 322                          */
 323                         if (++try_loading_module == 1)
 324                                 request_module("net-pf-%d-proto-%d-type-%d",
 325                                                PF_INET, protocol, sock->type);
 326                         /*
 327                          * Fall back to generic, e.g. net-pf-2-proto-132
 328                          * (net-pf-PF_INET-proto-IPPROTO_SCTP)
 329                          */
 330                         else
 331                                 request_module("net-pf-%d-proto-%d",
 332                                                PF_INET, protocol);
 333                         goto lookup_protocol;
 334                 } else
 335                         goto out_rcu_unlock;
 336         }
 337 
 338         err = -EPERM;
 339         if (sock->type == SOCK_RAW && !kern &&
 340             !ns_capable(net->user_ns, CAP_NET_RAW))
 341                 goto out_rcu_unlock;
 342 
 343         sock->ops = answer->ops;
 344         answer_prot = answer->prot;
 345         answer_no_check = answer->no_check;
 346         answer_flags = answer->flags;
 347         rcu_read_unlock();
 348 
 349         WARN_ON(answer_prot->slab == NULL);
 350 
 351         err = -ENOBUFS;
 352         sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot);
 353         if (sk == NULL)
 354                 goto out;
 355 
 356         err = 0;
 357         sk->sk_no_check = answer_no_check;
 358         if (INET_PROTOSW_REUSE & answer_flags)
 359                 sk->sk_reuse = SK_CAN_REUSE;
 360 
 361         inet = inet_sk(sk);
 362         inet->is_icsk = (INET_PROTOSW_ICSK & answer_flags) != 0;
 363 
 364         inet->nodefrag = 0;
 365 
 366         if (SOCK_RAW == sock->type) {
 367                 inet->inet_num = protocol;
 368                 if (IPPROTO_RAW == protocol)
 369                         inet->hdrincl = 1;
 370         }
 371 
 372         if (ipv4_config.no_pmtu_disc)
 373                 inet->pmtudisc = IP_PMTUDISC_DONT;
 374         else
 375                 inet->pmtudisc = IP_PMTUDISC_WANT;
 376 
 377         inet->inet_id = 0;
 378 
 379         sock_init_data(sock, sk);
 380 
 381         sk->sk_destruct    = inet_sock_destruct;
 382         sk->sk_protocol    = protocol;
 383         sk->sk_backlog_rcv = sk->sk_prot->backlog_rcv;
 384 
 385         inet->uc_ttl    = -1;
 386         inet->mc_loop   = 1;
 387         inet->mc_ttl    = 1;
 388         inet->mc_all    = 1;
 389         inet->mc_index  = 0;
 390         inet->mc_list   = NULL;
 391         inet->rcv_tos   = 0;
 392 
 393         sk_refcnt_debug_inc(sk);
 394 
 395         if (inet->inet_num) {
 396                 /* It assumes that any protocol which allows
 397                  * the user to assign a number at socket
 398                  * creation time automatically
 399                  * shares.
 400                  */
 401                 inet->inet_sport = htons(inet->inet_num);
 402                 /* Add to protocol hash chains. */
 403                 sk->sk_prot->hash(sk);
 404         }
 405 
 406         if (sk->sk_prot->init) {
 407                 err = sk->sk_prot->init(sk);
 408                 if (err)
 409                         sk_common_release(sk);
 410         }
 411 out:
 412         return err;
 413 out_rcu_unlock:
 414         rcu_read_unlock();
 415         goto out;
 416 }
```

#### `sk_alloc`
```c
1329 /**
1330  *      sk_alloc - All socket objects are allocated here
1331  *      @net: the applicable net namespace
1332  *      @family: protocol family
1333  *      @priority: for allocation (%GFP_KERNEL, %GFP_ATOMIC, etc)
1334  *      @prot: struct proto associated with this new sock instance
1335  */
1336 struct sock *sk_alloc(struct net *net, int family, gfp_t priority,
1337                       struct proto *prot)
1338 {
1339         struct sock *sk;
1340 
1341         sk = sk_prot_alloc(prot, priority | __GFP_ZERO, family);
1342         if (sk) {
1343                 sk->sk_family = family;
1344                 /*
1345                  * See comment in struct sock definition to understand
1346                  * why we need sk_prot_creator -acme
1347                  */
1348                 sk->sk_prot = sk->sk_prot_creator = prot;
1349                 sock_lock_init(sk);
1350                 sock_net_set(sk, get_net(net));
1351                 atomic_set(&sk->sk_wmem_alloc, 1);
1352 
1353                 sock_update_classid(sk);
1354                 sock_update_netprioidx(sk);
1355         }
1356 
1357         return sk;
1358 }
1359 EXPORT_SYMBOL(sk_alloc);
```

```c
1246 
1247 static struct sock *sk_prot_alloc(struct proto *prot, gfp_t priority,
1248                 int family)
1249 {
1250         struct sock *sk;
1251         struct kmem_cache *slab;
1252 
1253         slab = prot->slab;
1254         if (slab != NULL) {
1255                 sk = kmem_cache_alloc(slab, priority & ~__GFP_ZERO);
1256                 if (!sk)
1257                         return sk;
1258                 if (priority & __GFP_ZERO) {
1259                         if (prot->clear_sk)
1260                                 prot->clear_sk(sk, prot->obj_size);
1261                         else
1262                                 sk_prot_clear_nulls(sk, prot->obj_size);
1263                 }
1264         } else
1265                 sk = kmalloc(prot->obj_size, priority);
1266 
1267         if (sk != NULL) {
1268                 kmemcheck_annotate_bitfield(sk, flags);
1269 
1270                 if (security_sk_alloc(sk, family, priority))
1271                         goto out_free;
1272 
1273                 if (!try_module_get(prot->owner))
1274                         goto out_free_sec;
1275                 sk_tx_queue_clear(sk);
1276         }
1277 
1278         return sk;
1279 
1280 out_free_sec:
1281         security_sk_free(sk);
1282 out_free:
1283         if (slab != NULL)
1284                 kmem_cache_free(slab, sk);
1285         else
1286                 kfree(sk);
1287         return NULL;
1288 }
1289 
```

for ipv4/tcp socket, here `prot` is `tcp_prot`;
```c
117 /** struct inet_sock - representation of INET sockets
118  *
119  * @sk - ancestor class
120  * @pinet6 - pointer to IPv6 control block
121  * @inet_daddr - Foreign IPv4 addr
122  * @inet_rcv_saddr - Bound local IPv4 addr
123  * @inet_dport - Destination port
124  * @inet_num - Local port
125  * @inet_saddr - Sending source
126  * @uc_ttl - Unicast TTL
127  * @inet_sport - Source port
128  * @inet_id - ID counter for DF pkts
129  * @tos - TOS
130  * @mc_ttl - Multicasting TTL
131  * @is_icsk - is this an inet_connection_sock?
132  * @uc_index - Unicast outgoing device index
133  * @mc_index - Multicast device index
134  * @mc_list - Group array
135  * @cork - info to build ip hdr on each ip frag while socket is corked
136  */
137 struct inet_sock {
138         /* sk and pinet6 has to be the first two members of inet_sock */
139         struct sock             sk;
140 #if IS_ENABLED(CONFIG_IPV6) 
141         struct ipv6_pinfo       *pinet6;
142 #endif   
143         /* Socket demultiplex comparisons on incoming packets. */
144 #define inet_daddr              sk.__sk_common.skc_daddr
145 #define inet_rcv_saddr          sk.__sk_common.skc_rcv_saddr
146 #define inet_addrpair           sk.__sk_common.skc_addrpair
147 #define inet_dport              sk.__sk_common.skc_dport
148 #define inet_num                sk.__sk_common.skc_num
149 #define inet_portpair           sk.__sk_common.skc_portpair
150 
151         __be32                  inet_saddr;
152         __s16                   uc_ttl;
153         __u16                   cmsg_flags;
154         __be16                  inet_sport;
155         __u16                   inet_id;
156 
157         struct ip_options_rcu __rcu     *inet_opt;
158         int                     rx_dst_ifindex;
159         __u8                    tos;
160         __u8                    min_ttl;
161         __u8                    mc_ttl;
162         __u8                    pmtudisc;
163         __u8                    recverr:1,
164                                 is_icsk:1,
165                                 freebind:1,
166                                 hdrincl:1,
167                                 mc_loop:1,
168                                 transparent:1,
169                                 mc_all:1,
170                                 nodefrag:1;
171         __u8                    rcv_tos;
172         int                     uc_index;
173         int                     mc_index;
174         __be32                  mc_addr;
175         struct ip_mc_socklist __rcu     *mc_list;
176         struct inet_cork_full   cork;
177 };
```

---
layout: post
title: "Add or Udpate Xfrm Policy"
date: 2012-11-10 00:00
comments: true
categories: [xfrm]
tags: [kernel, xfrm, SP]
---

xfrm policy add/del/update 是通过netlink消息进行的。 其中xfrm_add_policy 用来添加 xfrm policy。

<!-- more -->
##netlink message type
```
163         XFRM_MSG_NEWPOLICY,
164 #define XFRM_MSG_NEWPOLICY XFRM_MSG_NEWPOLICY
165         XFRM_MSG_DELPOLICY,
166 #define XFRM_MSG_DELPOLICY XFRM_MSG_DELPOLICY
167         XFRM_MSG_GETPOLICY,
168 #define XFRM_MSG_GETPOLICY XFRM_MSG_GETPOLICY
```
```
2290         [XFRM_MSG_NEWPOLICY   - XFRM_MSG_BASE] = { .doit = xfrm_add_policy    },
2291         [XFRM_MSG_DELPOLICY   - XFRM_MSG_BASE] = { .doit = xfrm_get_policy    },
2292         [XFRM_MSG_GETPOLICY   - XFRM_MSG_BASE] = { .doit = xfrm_get_policy,
...
2298         [XFRM_MSG_UPDPOLICY   - XFRM_MSG_BASE] = { .doit = xfrm_add_policy    },
```
```
 501 struct xfrm_policy {
 502 #ifdef CONFIG_NET_NS
 503         struct net              *xp_net;
 504 #endif
 505         struct hlist_node       bydst;
 506         struct hlist_node       byidx;
 507
 508         /* This lock only affects elements except for entry. */
 509         rwlock_t                lock;
 510         atomic_t                refcnt;
 511         struct timer_list       timer;
 512
 513         struct flow_cache_object flo;
 514         atomic_t                genid;
 515         u32                     priority;
 516         u32                     index;
 517         struct xfrm_mark        mark;
 518         struct xfrm_selector    selector;
 519         struct xfrm_lifetime_cfg lft;
 520         struct xfrm_lifetime_cur curlft;
 521         struct xfrm_policy_walk_entry walk;
 522         u8                      type;
 523         u8                      action;
 524         u8                      flags;
 525         u8                      xfrm_nr;
 526         u16                     family;
 527         struct xfrm_sec_ctx     *security;
 528         struct xfrm_tmpl        xfrm_vec[XFRM_MAX_DEPTH];
 529 };
```
##Add a policy:
```
> xfrm_add_policy
> > verify_newpolicy_info
> > xfrm_policy_construct
> > xfrm_policy_insert
> > > policy_hash_bysel
> > > hlist_for_each_entry
> > > match ? hlist_add_after or hlist_add_head
> > > hlist_add_head
> > > mod_timer
> > > list_add(&policy->walk.all, &net->xfrm.policy_all);
> > > delpol ? xfrm_policy_kill(delpol);
> > > xfrm_bydst_should_resize? schedule_work(&net->xfrm.policy_hash_work);
> > km_policy_notify
> > xfrm_pol_put
```
```
1363 static int xfrm_add_policy(struct sk_buff *skb, struct nlmsghdr *nlh,
1364                 struct nlattr **attrs)
1365 {
1366         struct net *net = sock_net(skb->sk);
1367         struct xfrm_userpolicy_info *p = nlmsg_data(nlh);
1368         struct xfrm_policy *xp;
1369         struct km_event c;
1370         int err;
1371         int excl;
1372         uid_t loginuid = audit_get_loginuid(current);
1373         u32 sessionid = audit_get_sessionid(current);
1374         u32 sid;
1375
1376         err = verify_newpolicy_info(p);
1377         if (err)
1378                 return err;
1379         err = verify_sec_ctx_len(attrs);
1380         if (err)
1381                 return err;
1382
1383         xp = xfrm_policy_construct(net, p, attrs, &err); <== construct a xfrm policy with infmation from user space.
1384         if (!xp)
1385                 return err;
1386
1387         /* shouldn't excl be based on nlh flags??
1388          * Aha! this is anti-netlink really i.e  more pfkey derived
1389          * in netlink excl is a flag and you wouldnt need
1390          * a type XFRM_MSG_UPDPOLICY - JHS */
1391         excl = nlh->nlmsg_type == XFRM_MSG_NEWPOLICY;
1392         err = xfrm_policy_insert(p->dir, xp, excl);  <== insert/update a xfrm policy.
1393         security_task_getsecid(current, &sid);
1394         xfrm_audit_policy_add(xp, err ? 0 : 1, loginuid, sessionid, sid);
1395
1396         if (err) {
1397                 security_xfrm_policy_free(xp->security);
1398                 kfree(xp);
1399                 return err;
1400         }
1401
1402         c.event = nlh->nlmsg_type;
1403         c.seq = nlh->nlmsg_seq;
1404         c.pid = nlh->nlmsg_pid;
1405         km_policy_notify(xp, p->dir, &c);  <==== send notify message to KM.
1406
1407         xfrm_pol_put(xp);
1408
1409         return 0;
1410 }
```

xfrm policy is a hash list array, every element is a hist list. In the list the policies are sorted with increased policy’s priority.
```
 548 int xfrm_policy_insert(int dir, struct xfrm_policy *policy, int excl)
 549 {
 550         struct net *net = xp_net(policy);
 551         struct xfrm_policy *pol;
 552         struct xfrm_policy *delpol;
 553         struct hlist_head *chain;
 554         struct hlist_node *entry, *newpos;
 555         u32 mark = policy->mark.v & policy->mark.m;
 556
 557         write_lock_bh(&xfrm_policy_lock);  <== use rw lock to protect xfrm polciy database.
 558         chain = policy_hash_bysel(net, &policy->selector, policy->family, dir);
 559         delpol = NULL;
 560         newpos = NULL;
 561         hlist_for_each_entry(pol, entry, chain, bydst) {
 562                 if (pol->type == policy->type &&
 563                     !selector_cmp(&pol->selector, &policy->selector) &&
 564                     (mark & pol->mark.m) == pol->mark.v &&
 565                     xfrm_sec_ctx_match(pol->security, policy->security) &&
 566                     !WARN_ON(delpol)) {
 567                         if (excl) {
 568                                 write_unlock_bh(&xfrm_policy_lock);
 569                                 return -EEXIST;
 570                         }
 571                         delpol = pol;
 572                         if (policy->priority > pol->priority)   <====??? todo
 573                                 continue;
 574                 } else if (policy->priority >= pol->priority) {
 575                         newpos = &pol->bydst;
 576                         continue;
 577                 }
 578                 if (delpol)
 579                         break;
 580         }
 581         if (newpos)
 582                 hlist_add_after(newpos, &policy->bydst);
 583         else
 584                 hlist_add_head(&policy->bydst, chain);
 585         xfrm_pol_hold(policy);
 586         net->xfrm.policy_count[dir]++;
 587         atomic_inc(&flow_cache_genid); <== it will be used by flow cache.
 588         if (delpol)
 589                 __xfrm_policy_unlink(delpol, dir);
 590         policy->index = delpol ? delpol->index : xfrm_gen_index(net, dir);
 591         hlist_add_head(&policy->byidx, net->xfrm.policy_byidx+idx_hash(net, policy->index));
 592         policy->curlft.add_time = get_seconds();
 593         policy->curlft.use_time = 0;
 594         if (!mod_timer(&policy->timer, jiffies + HZ))
 595                 xfrm_pol_hold(policy);
 596         list_add(&policy->walk.all, &net->xfrm.policy_all);
 597         write_unlock_bh(&xfrm_policy_lock);
 598
 599         if (delpol)
 600                 xfrm_policy_kill(delpol);
 601         else if (xfrm_bydst_should_resize(net, dir, NULL))
 602                 schedule_work(&net->xfrm.policy_hash_work);
 603
 604         return 0;
 605 }
 606 EXPORT_SYMBOL(xfrm_policy_insert);
 ```

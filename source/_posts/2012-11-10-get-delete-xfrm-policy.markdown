---
layout: post
title: "Get or Delete Xfrm Policy"
date: 2012-11-10 00:00
comments: true
categories: [xfrm] 
tags: [xfrm, kernel, policy]
---

##summary
xfrm_get_policy first locate the xfrm policy by policy index(from user space) or policy selector.

1. if get_policy, alloc a new skb, and encapsulate the xfrm policy to it, then sent  it. 
2. if delete policy, call `xfrm_audit_policy_delete` to delete the plolicy, and call km_policy_notify to notify.

xfrm policy del/get 使用的是同一个函数 `xfrm_get_policy`.
<!-- more -->

```c 
2291         [XFRM_MSG_DELPOLICY - XFRM_MSG_BASE] = { .doit = xfrm_get_policy    },
2292         [XFRM_MSG_GETPOLICY - XFRM_MSG_BASE] = { .doit = xfrm_get_policy,
```
##data struct
```
405 struct xfrm_userpolicy_id {
406         struct xfrm_selector            sel;
407         __u32                           index;
408         __u8                            dir;
409 };
```

##functions
```
1567 static int xfrm_get_policy(struct sk_buff *skb, struct nlmsghdr *nlh,
1568                 struct nlattr **attrs)
1569 {
1570         struct net *net = sock_net(skb->sk);
1571         struct xfrm_policy *xp;
1572         struct xfrm_userpolicy_id *p;
1573         u8 type = XFRM_POLICY_TYPE_MAIN;
1574         int err;
1575         struct km_event c;
1576         int delete;
1577         struct xfrm_mark m;
1578         u32 mark = xfrm_mark_get(attrs, &m);
1579
1580         p = nlmsg_data(nlh);
1581         delete = nlh->nlmsg_type == XFRM_MSG_DELPOLICY;
1582
1583         err = copy_from_user_policy_type(&type, attrs);
1584         if (err)
1585                 return err;
1586
1587         err = verify_policy_dir(p->dir);
1588         if (err)
1589                 return err;
1590/*start find the policy by index or by selector.*/
1591         if (p->index)
1592                 xp = xfrm_policy_byid(net, mark, type, p->dir, p->index, delete, &err);
1593         else {
1594                 struct nlattr *rt = attrs[XFRMA_SEC_CTX];
1595                 struct xfrm_sec_ctx *ctx;
1596
1597                 err = verify_sec_ctx_len(attrs);
1598                 if (err)
1599                         return err;
1600
1601                 ctx = NULL;
1602                 if (rt) {
1603                         struct xfrm_user_sec_ctx *uctx = nla_data(rt);
1604
1605                         err = security_xfrm_policy_alloc(&ctx, uctx);
1606                         if (err)
1607                                 return err;
1608                 }
1609                 xp = xfrm_policy_bysel_ctx(net, mark, type, p->dir, &p->sel,
1610                                            ctx, delete, &err);
1611                 security_xfrm_policy_free(ctx);
1612         }
1613         if (xp == NULL)
1614                 return -ENOENT;
1615
1616         if (!delete) { <== 'get': encaplate to skb and sent it by netlink
1617                 struct sk_buff *resp_skb;
1618
1619                 resp_skb = xfrm_policy_netlink(skb, xp, p->dir, nlh->nlmsg_seq);
1620                 if (IS_ERR(resp_skb)) {
1621                         err = PTR_ERR(resp_skb);
1622                 } else {
1623                         err = nlmsg_unicast(net->xfrm.nlsk, resp_skb,
1624                                             NETLINK_CB(skb).pid);
1625                 }
1626         } else { <== delete xfrm policy and sent a notify netlink message.
1627                 uid_t loginuid = audit_get_loginuid(current);
1628                 u32 sessionid = audit_get_sessionid(current);
1629                 u32 sid;
1630
1631                 security_task_getsecid(current, &sid);
1632                 xfrm_audit_policy_delete(xp, err ? 0 : 1, loginuid, sessionid,
1633                                          sid);
1634
1635                 if (err != 0)
1636                         goto out;
1637
1638                 c.data.byid = p->index;
1639                 c.event = nlh->nlmsg_type;
1640                 c.seq = nlh->nlmsg_seq;
1641                 c.pid = nlh->nlmsg_pid;
1642                 km_policy_notify(xp, p->dir, &c);
1643         }
1644
1645 out:
1646         xfrm_pol_put(xp);
1647         return err;
1648 }
```

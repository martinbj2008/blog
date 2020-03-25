---
layout: post
title: "xfrm in kernel"
date: 2009-05-13 00:00
comments: true
categories: [xfrm]
tags: [kenel, xfrm]
---

### global var and structure:
`static DEFINE_PER_NET(struct hlist_head *, xfrm_state_byspi);`     
`static struct xfrm_policy_afinfo *xfrm_policy_afinfo[NPROTO];`     
xfrm_policy_afinfo 定义一个大的数组，每一个元素对应一个地址族，如ipv4（AF_INET），ipv6(AF_INET6).

<!-- more -->

```c
struct xfrm_policy_afinfo { 
    unsigned short        family; 
    rwlock_t        lock; 
    struct xfrm_type_map    *type_map; 
    struct dst_ops        *dst_ops; 
    void            (*garbage_collect)(net_t net); 
    int            (*dst_lookup)(struct xfrm_dst **dst, struct flowi *fl); 
    int            (*get_saddr)(net_t net, xfrm_address_t *saddr, xfrm_address_t *daddr); 
    struct dst_entry    *(*find_bundle)(struct flowi *fl, struct xfrm_policy *policy); 
    int            (*bundle_create)(struct xfrm_policy *policy, 
                        struct xfrm_state **xfrm, 
                        int nx, 
                        struct flowi *fl, 
                        struct dst_entry **dst_p); 
    void            (*decode_session)(struct sk_buff *skb, 
                        struct flowi *fl); 
}; 
```

在struct xfrm_policy_afinfo中有一个元素
```c
    struct xfrm_type_map    *type_map;
struct xfrm_type_map { 
    rwlock_t        lock; 
    struct xfrm_type    *map[256]; 
};
```

map 也是一个指针数组，其每个元素对应一个应用层的协议如 ESP(IPPROTO_ESP), AH, UDP(IPPROTO_UDP),TCP等。

两个相关的注册函数：

关于xfrm_policy_afinfo
```c
int xfrm_policy_register_afinfo(struct xfrm_policy_afinfo *afinfo)；

static void __init xfrm4_policy_init(void) 
{ 
    xfrm_policy_register_afinfo(&xfrm4_policy_afinfo); 
} 

static void __init xfrm6_policy_init(void) 
{ 
    xfrm_policy_register_afinfo(&xfrm6_policy_afinfo); 
} 
```

```c
int xfrm_policy_register_afinfo(struct xfrm_policy_afinfo *afinfo) 
{ 
    int err = 0; 
    if (unlikely(afinfo == NULL)) 
        return -EINVAL; 
    if (unlikely(afinfo->family >= NPROTO)) 
        return -EAFNOSUPPORT; 
    write_lock(&xfrm_policy_afinfo_lock); 
    if (unlikely(xfrm_policy_afinfo[afinfo->family] != NULL)) 
        err = -ENOBUFS; 
    else { 
        struct dst_ops *dst_ops = afinfo->dst_ops; 
        if (likely(dst_ops->kmem_cachep == NULL)) 
            dst_ops->kmem_cachep = xfrm_dst_cache; 
        if (likely(dst_ops->check == NULL)) 
            dst_ops->check = xfrm_dst_check; 
        if (likely(dst_ops->negative_advice == NULL)) 
            dst_ops->negative_advice = xfrm_negative_advice; 
        if (likely(dst_ops->link_failure == NULL)) 
            dst_ops->link_failure = xfrm_link_failure; 
        if (likely(afinfo->garbage_collect == NULL)) 
            afinfo->garbage_collect = __xfrm_garbage_collect; 
        xfrm_policy_afinfo[afinfo->family] = afinfo; 
    } 
    write_unlock(&xfrm_policy_afinfo_lock); 
    return err; 
} 
```


```c
int xfrm_register_type(struct xfrm_type *type, unsigned short family) 
{ 
    struct xfrm_policy_afinfo *afinfo = xfrm_policy_get_afinfo(family); 
    struct xfrm_type_map *typemap; 
    int err = 0; 

    if (unlikely(afinfo == NULL)) 
        return -EAFNOSUPPORT; 
    typemap = afinfo->type_map; 

    write_lock(&typemap->lock); 
    if (likely(typemap->map[type->proto] == NULL)) 
        typemap->map[type->proto] = type; 
    else 
        err = -EEXIST; 
    write_unlock(&typemap->lock); 
    xfrm_policy_put_afinfo(afinfo); 
    return err; 
}
```


---
layout: post
title: "register_pernet_subsys 笔记"
date: 2011-02-25 00:00
comments: true
categories: [netcore]
tags: [kernel, pernet]
---

### pernet ops
```c
/**
 *      register_pernet_subsys - register a network namespace subsystem
 *    @ops:  pernet operations structure for the subsystem
 *
 *    Register a subsystem which has init and exit functions
 *    that are called when network namespaces are created and
 *    destroyed respectively.
 *
 *    When registered all network namespace init functions are
 *    called for every existing network namespace.  Allowing kernel
 *    modules to have a race free view of the set of network namespaces.
 *
 *    When a new network namespace is created all of the init
 *    methods are called in the order in which they were registered.
 *
 *    When a network namespace is destroyed all of the exit methods
 *    are called in the reverse of the order with which they were
 *    registered.
 */
```


```c
int register_pernet_subsys(struct pernet_operations *ops)
{
    int error;
    mutex_lock(&net_mutex);
    error =  register_pernet_operations(first_device, ops);
    mutex_unlock(&net_mutex);
    return error;
}
===>static int register_pernet_operations(struct list_head *list,
                  struct pernet_operations *ops)
{
    error = __register_pernet_operations(list, ops);
}

======>#ifdef CONFIG_NET_NS
static int __register_pernet_operations(struct list_head *list,
                    struct pernet_operations *ops)
    LIST_HEAD(net_exit_list);

    list_add_tail(&ops->list, list);
    if (ops->init || (ops->id && ops->size)) {
        for_each_net(net) {
            error = ops_init(ops, net);
            if (error)
                goto out_undo;
            list_add_tail(&net->exit_list, &net_exit_list);
        }
    }
    return 0;


========>#ifdef CONFIG_NET_NS
    static int __register_pernet_operations(struct list_head *list,
                    struct pernet_operations *ops)
    struct net *net;
    int error;
    LIST_HEAD(net_exit_list);
 
    list_add_tail(&ops->list, list);
    if (ops->init || (ops->id && ops->size)) {
        for_each_net(net) {
=============>            error = ops_init(ops, net);
            if (error)
                goto out_undo;
=============>        list_add_tail(&net->exit_list, &net_exit_list);<<< confused?!!! net_exit_list局部变量？

        }
    }
    return 0;


=============>static int ops_init(const struct pernet_operations *ops, struct net *net)
{
    int err;
    if (ops->id && ops->size) {
        void *data = kzalloc(ops->size, GFP_KERNEL);
        if (!data)
            return -ENOMEM;
 
        err = net_assign_generic(net, *ops->id, data);
        if (err) {
            kfree(data);
            return err;
        }
    }
    if (ops->init)
        return ops->init(net);<====== the ops->init will be called.
    return 0;
}
```
 
 
####Fox example
inet6_init in pernet.

```c
static struct pernet_operations inet6_net_ops = { 
    .init = inet6_net_init, 
    .exit = inet6_net_exit, 
};
static int __init inet6_init(void) 
{ 
.....
    err = register_pernet_subsys(&inet6_net_ops); 
    if (err) 
        goto register_pernet_fail; 
.....
}
call:     ops->init(net);<====== the ops->init will be called.
equal with =======   .init = inet6_net_init,
                     inet6_net_init(net);

```

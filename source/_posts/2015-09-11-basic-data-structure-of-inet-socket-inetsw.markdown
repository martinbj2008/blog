---
layout: post
title: "basic data structure of inet socket: inetsw"
date: 2015-09-11 10:41:11 +0800
comments: true
categories: [socket]
tags: [socket, inet]
---

### protocol链表数组
inet有些例外，因为inet支持的类型太过复杂(maybe), 所以引入了一个inetsw的链表数组。
就像注释里说的一样。inetsw是inet socket创建的基础，包含创建inet socket全部的所需要信息。
#### `inetsw`
```
 125 /* The inetsw table contains everything that inet_create needs to
 126  * build a new socket.
 127  */
 128 static struct list_head inetsw[SOCK_MAX];
 129 static DEFINE_SPINLOCK(inetsw_lock);
```

`inetsw`是一个链表头的数据，每个链表是具有相同的type的, 具体见[socket type](#socketType).    
每个节点是一个`struct inet_protosw`. 每个节点是通过[`net_register_protosw`](#registerProtoNode)
插入到其type对应的链表里的。
<!-- more -->

#### `struct inet_protosw`
```
 78 /* This is used to register socket interfaces for IP protocols.  */
 79 struct inet_protosw {
 80         struct list_head list;
 81
 82         /* These two fields form the lookup key.  */
 83         unsigned short   type;     /* This is the 2nd argument to socket(2). */
 84         unsigned short   protocol; /* This is the L4 protocol number.  */
 85
 86         struct proto     *prot;
 87         const struct proto_ops *ops;
 88
 89         unsigned char    flags;      /* See INET_PROTOSW_* below.  */
 90 };
```

##### 两个重要的指针
每种socket都有其对应的`struct proto_ops *ops`,该指针会存放在`struct socket`里。  
inet的socket在创建的时候会指定两个函数指针:    
1. `struct proto_ops *ops`   
	该指针被存放到`struct socket`里.   
	一般socket这些函数就是真正的bind, connect,sendmsg函数。   
	但是在inet里，这些函数其实是通用的或者基于某个type(如stream)的操作。    
	如`inet_bind`, `inet_sendmsg` `inet_stream_connect`等。   
2. `struct proto *prot`
	该指针被存放到`struct sock`里.   
	各个协议自己的特殊处理，被抽象并整理到这里，如`tcp_v4_connect`.   
	`struct proto_ops *ops`经常使用'sk->sk_prot->XXX`调用响应的proto处理函数。    
	如：`inet_stream_connect/__inet_stream_connect` 调用｀sk->sk_prot->connect`

##### 一个具体的例子
如tcp协议的`struct proto_ops *ops`是`inet_stream_ops`, `struct proto *prot`是`tcp_prot`
```
 997         {
 998                 .type =       SOCK_STREAM,
 999                 .protocol =   IPPROTO_TCP,
1000                 .prot =       &tcp_prot,
1001                 .ops =        &inet_stream_ops,
1002                 .flags =      INET_PROTOSW_PERMANENT |
1003                               INET_PROTOSW_ICSK,
1004         },
```

##### 两个指针何时被初始化的
上述两个指针在sock创建时候就被确定下来了.   
函数`inet_create`会根据socket系统调用指定的type，proto参数，去inetsw链表里查找。   
并将找到的节点里的`struct proto_ops *ops`和`struct proto *prot`分别保存到该socket对应的`socket`和`sock`里。

<span id="registerProtoNode">

##### 几点说明 `struct inet_protosw`
1. permanent protocol
因为`struct inet_protosw`有永久(permanent)和临时之分。

所为permanent节点就是其结构体的flag带有`INET_PROTOSW_PERMANENT`标志位。说明其对应的协议不允许被override.
而临时节点则相反，没有`INET_PROTOSW_PERMANENT`标志，新插入的协议override原有的。

所有永久的节点被放在了链表的头部，而新插入的临时节点
都会被放到permanent节点的后面。

```
1030
1031 #define INETSW_ARRAY_LEN ARRAY_SIZE(inetsw_array)
1032
1033 void inet_register_protosw(struct inet_protosw *p)
1034 {
1035         struct list_head *lh;
1036         struct inet_protosw *answer;
1037         int protocol = p->protocol;
1038         struct list_head *last_perm;
1039
1040         spin_lock_bh(&inetsw_lock);
1041
1042         if (p->type >= SOCK_MAX)
1043                 goto out_illegal;
1044
1045         /* If we are trying to override a permanent protocol, bail. */
1046         last_perm = &inetsw[p->type];
1047         list_for_each(lh, &inetsw[p->type]) {
1048                 answer = list_entry(lh, struct inet_protosw, list);
1049                 /* Check only the non-wild match. */
1050                 if ((INET_PROTOSW_PERMANENT & answer->flags) == 0)
1051                                 break;
1052                 if (protocol == answer->protocol)
1053                         goto out_permanent;
1054                 last_perm = lh;
1055         }
1056
1057         /* Add the new entry after the last permanent entry if any, so that
1058          * the new entry does not override a permanent entry when matched with
1059          * a wild-card protocol. But it is allowed to override any existing
1060          * non-permanent entry.  This means that when we remove this entry, the
1061          * system automatically returns to the old behavior.
1062          */
1063         list_add_rcu(&p->list, last_perm);
1064 out:
1065         spin_unlock_bh(&inetsw_lock);
1066
1067         return;
1068
1069 out_permanent:
1070         pr_err("Attempt to override permanent protocol %d\n", protocol);
1071         goto out;
1072
1073 out_illegal:
1074         pr_err("Ignoring attempt to register invalid socket type %d\n",
1075                p->type);
1076         goto out;
1077 }
1078 EXPORT_SYMBOL(inet_register_protosw);
```
</span>

<span id="socketType">
##### `enum socket_type`
```
 58 enum sock_type {
 59         SOCK_STREAM     = 1,
 60         SOCK_DGRAM      = 2,
 61         SOCK_RAW        = 3,
 62         SOCK_RDM        = 4,
 63         SOCK_SEQPACKET  = 5,
 64         SOCK_DCCP       = 6,
 65         SOCK_PACKET     = 10,
 66 };
```
</span>

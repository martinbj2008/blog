---
layout: post
title: "pf_key module summary"
date: 2010-06-02 00:00
comments: true
categories: [xfrm]
tags: [kernel, xfrm]
---

###af_key.c
linux kernel provide 3 method to manager SA/SP,
such as add/del/flush/dump SAs/SPs.
1. pf_key socket.
2. netlink message.
3. socket option.

The af_key.c implement the pf_key socket.

###part 1. pf_key socket defination about socket opertion.
important function is 
pfkey_create,pfkey_sendmsg,pfkey_recvmsg,
pfkey_release,datagram_poll, 


```c
static const struct proto_ops pfkey_ops = { 
.family        =    PF_KEY, 
.owner        =    THIS_MODULE, 
/* Operations that make no sense on pfkey sockets. */ 
.bind        =    sock_no_bind, 
.connect    =    sock_no_connect, 
.socketpair    =    sock_no_socketpair, 
.accept        =    sock_no_accept, 
.getname    =    sock_no_getname, 
.ioctl        =    sock_no_ioctl, 
.listen        =    sock_no_listen, 
.shutdown    =    sock_no_shutdown, 
.setsockopt    =    sock_no_setsockopt, 
.getsockopt    =    sock_no_getsockopt, 
.mmap        =    sock_no_mmap, 
.sendpage    =    sock_no_sendpage, 

/* Now the operations that really occur. */ 
.release    =    pfkey_release, 
.poll        =    datagram_poll, 
.sendmsg    =    pfkey_sendmsg, 
.recvmsg    =    pfkey_recvmsg, 
};


static struct net_proto_family pfkey_family_ops = { 
.family    =    PF_KEY, 
.create    =    pfkey_create, 
.owner    =    THIS_MODULE, 
};


struct pfkey_sock { 
/* struct sock must be the first member of struct pfkey_sock */ 
struct sock    sk; 
int        registered; 
int        promisc; 

struct { 
uint8_t        msg_version; 
uint32_t    msg_pid; 
int        (*dump)(struct pfkey_sock *sk); 
void        (*done)(struct pfkey_sock *sk); 
union { 
struct xfrm_policy_walk    policy; 
struct xfrm_state_walk    state; 
} u; 
struct sk_buff    *skb; 
} dump; 
}; 
```


###part 2. pf_key kernel message 

```c
static struct xfrm_mgr pfkeyv2_mgr =
{ 
.id        = "pfkeyv2", 
.notify        = pfkey_send_notify, 
.acquire    = pfkey_send_acquire, 
.compile_policy    = pfkey_compile_policy, 
.new_mapping    = pfkey_send_new_mapping, 
.notify_policy    = pfkey_send_policy_notify, 
.migrate    = pfkey_send_migrate, 
};
```

### pf_key message process.

in kernel 3.0, pf_key message format
A traditional TLV format. 

`header + (extenion-header + extention_value)*n`

The header is sadb_msg.
extention header is sadb_ext.
extention value is different according the extention header.
Such as sadb_sa,sadb_x_policy and so on.

```c
struct sadb_msg { 
uint8_t        sadb_msg_version; 
uint8_t        sadb_msg_type; 
uint8_t        sadb_msg_errno; 
uint8_t        sadb_msg_satype; 
uint16_t    sadb_msg_len; 
uint16_t    sadb_msg_reserved; 
uint32_t    sadb_msg_seq; 
uint32_t    sadb_msg_pid; 
} __attribute__((packed)); 
/* sizeof(struct sadb_msg) == 16 */ 

struct sadb_ext { 
uint16_t    sadb_ext_len; 
uint16_t    sadb_ext_type; 
} __attribute__((packed)); 
/* sizeof(struct sadb_ext) == 4 */ 


struct sadb_sa { 
uint16_t    sadb_sa_len; 
uint16_t    sadb_sa_exttype; 
__be32        sadb_sa_spi; 
uint8_t        sadb_sa_replay; 
uint8_t        sadb_sa_state; 
uint8_t        sadb_sa_auth; 
uint8_t        sadb_sa_encrypt; 
uint32_t    sadb_sa_flags; 
} __attribute__((packed)); 
/* sizeof(struct sadb_sa) == 16 */

struct sadb_x_policy { 
uint16_t    sadb_x_policy_len; 
uint16_t    sadb_x_policy_exttype; 
uint16_t    sadb_x_policy_type; 
uint8_t        sadb_x_policy_dir; 
uint8_t        sadb_x_policy_reserved; 
uint32_t    sadb_x_policy_id; 
uint32_t    sadb_x_policy_priority; 
} __attribute__((packed)); 
/* sizeof(struct sadb_x_policy) == 16 */
```

The application program(such as setkey) sent a command to kernel by sendmsg system API.
Thus in kernel pf_key will call pfkey_sendmsg.
pfkey_sendmsg will call pfkey_get_base_msg to do some simple check, and 
then call pfkey_process.

pfkey_process will first pfkey_broadcast, then divid the extention message
to a pointer array one by one.
`void *ext_hdrs\[SADB_EXT_MAX\]; `
`SADB_EXT_SA`         ---->
`SADB_EXT_ADDRESS_SRC`---->
`SADB_EXT_ADDRESS_DST`---->
this pointer array will be used by the following handler.

and then call the pfkey_handler according the sadb_msg_type in the pf_key messag header.


`typedef int (*pfkey_handler)(struct sock *sk, struct sk_buff *skb, struct sadb_msg *hdr, void **ext_hdrs); `

```c
typedef int (*pfkey_handler)(struct sock *sk, struct sk_buff *skb, struct sadb_msg *hdr, void **ext_hdrs); 
static pfkey_handler pfkey_funcs[SADB_MAX + 1] = { 
[SADB_RESERVED]        = pfkey_reserved, 
[SADB_GETSPI]        = pfkey_getspi, 
[SADB_UPDATE]        = pfkey_add, 
[SADB_ADD]        = pfkey_add, 
[SADB_DELETE]        = pfkey_delete, 
[SADB_GET]        = pfkey_get, 
[SADB_ACQUIRE]        = pfkey_acquire, 
[SADB_REGISTER]        = pfkey_register, 
[SADB_EXPIRE]        = NULL, 
[SADB_FLUSH]        = pfkey_flush, 
[SADB_DUMP]        = pfkey_dump, 
[SADB_X_PROMISC]    = pfkey_promisc, 
[SADB_X_PCHANGE]    = NULL, 
[SADB_X_SPDUPDATE]    = pfkey_spdadd, 
[SADB_X_SPDADD]        = pfkey_spdadd, 
[SADB_X_SPDDELETE]    = pfkey_spddelete, 
[SADB_X_SPDGET]        = pfkey_spdget, 
[SADB_X_SPDACQUIRE]    = NULL, 
[SADB_X_SPDDUMP]    = pfkey_spddump, 
[SADB_X_SPDFLUSH]    = pfkey_spdflush, 
[SADB_X_SPDSETIDX]    = pfkey_spdadd, 
[SADB_X_SPDDELETE2]    = pfkey_spdget, 
[SADB_X_MIGRATE]    = pfkey_migrate, 
};
```

The policy related function was done in xfrm_policy.c and xfrm_state.c
3.1  policy add handler: pfkey_spdadd

3.2  polcy dump handler: pfkey_spddump
function pfkey_xfrm_policy2msg

3.3  policy flush handler: pfkey_spdflush.

3.4  SA add handler:pfkey_add

3.2  SA dump handler:pfkey_dump

3.3  SA flush handler:pfkey_flush

---
layout: post
title: "inetsw table"
date: 2013-11-22 11:34
comments: true
categories: 
---

### Data struct.
``` c
 123 /* The inetsw table contains everything that inet_create needs to
 124  * build a new socket.
 125  */
 126 static struct list_head inetsw[SOCK_MAX];
 127 static DEFINE_SPINLOCK(inetsw_lock);
```
<!-- more -->

Just like the `net_families`, spinlock is used for mutex of multi-writer
and rcu lock is used for reader/writer.

``` c
 73 /* This is used to register socket interfaces for IP protocols.  */
 74 struct inet_protosw {
 75         struct list_head list;
 76 
 77         /* These two fields form the lookup key.  */
 78         unsigned short   type;     /* This is the 2nd argument to socket(2). */
 79         unsigned short   protocol; /* This is the L4 protocol number.  */
 80 
 81         struct proto     *prot;
 82         const struct proto_ops *ops;
 83 
 84         char             no_check;   /* checksum on rcv/xmit/none? */
 85         unsigned char    flags;      /* See INET_PROTOSW_* below.  */
 86 };
 87 #define INET_PROTOSW_REUSE 0x01      /* Are ports automatically reusable? */
 88 #define INET_PROTOSW_PERMANENT 0x02  /* Permanent protocols are unremovable. */
 89 #define INET_PROTOSW_ICSK      0x04  /* Is this an inet_connection_sock? */
```

### `inet_register_protosw` and `inet_unregister_protosw`
``` c
1068 #define INETSW_ARRAY_LEN ARRAY_SIZE(inetsw_array)
1069 
1070 void inet_register_protosw(struct inet_protosw *p)
1071 {
1072         struct list_head *lh;
1073         struct inet_protosw *answer;
1074         int protocol = p->protocol;
1075         struct list_head *last_perm;
1076 
1077         spin_lock_bh(&inetsw_lock);
1078 
1079         if (p->type >= SOCK_MAX)
1080                 goto out_illegal;
1081 
1082         /* If we are trying to override a permanent protocol, bail. */
1083         answer = NULL;
1084         last_perm = &inetsw[p->type];
1085         list_for_each(lh, &inetsw[p->type]) {
1086                 answer = list_entry(lh, struct inet_protosw, list);
1087 
1088                 /* Check only the non-wild match. */
1089                 if (INET_PROTOSW_PERMANENT & answer->flags) {
1090                         if (protocol == answer->protocol)
1091                                 break;
1092                         last_perm = lh;
1093                 }
1094 
1095                 answer = NULL;
1096         }
1097         if (answer)
1098                 goto out_permanent;
1099 
1100         /* Add the new entry after the last permanent entry if any, so that
1101          * the new entry does not override a permanent entry when matched with
1102          * a wild-card protocol. But it is allowed to override any existing
1103          * non-permanent entry.  This means that when we remove this entry, the
1104          * system automatically returns to the old behavior.
1105          */
1106         list_add_rcu(&p->list, last_perm);
1107 out:
1108         spin_unlock_bh(&inetsw_lock);
1109 
1110         return;
1111 
1112 out_permanent:
1113         pr_err("Attempt to override permanent protocol %d\n", protocol);
1114         goto out;
1115 
1116 out_illegal:
1117         pr_err("Ignoring attempt to register invalid socket type %d\n",
1118                p->type);
1119         goto out;
1120 }
1121 EXPORT_SYMBOL(inet_register_protosw);
1122 
1123 void inet_unregister_protosw(struct inet_protosw *p)
1124 {
1125         if (INET_PROTOSW_PERMANENT & p->flags) {
1126                 pr_err("Attempt to unregister permanent protocol %d\n",
1127                        p->protocol);
1128         } else {
1129                 spin_lock_bh(&inetsw_lock);
1130                 list_del_rcu(&p->list);
1131                 spin_unlock_bh(&inetsw_lock);
1132 
1133                 synchronize_net();
1134         }
1135 }
1136 EXPORT_SYMBOL(inet_unregister_protosw);
```

### Register 4 kinds of inet socket.
There are 4 kinds of inet socket: `tcp`, `udp`, `icmp`, `raw`.
``` c
1025 /* Upon startup we insert all the elements in inetsw_array[] into
1026  * the linked list inetsw.
1027  */
1028 static struct inet_protosw inetsw_array[] =
1029 {
1030         {
1031                 .type =       SOCK_STREAM,
1032                 .protocol =   IPPROTO_TCP,
1033                 .prot =       &tcp_prot,
1034                 .ops =        &inet_stream_ops,
1035                 .no_check =   0,
1036                 .flags =      INET_PROTOSW_PERMANENT |
1037                               INET_PROTOSW_ICSK,
1038         },
1039 
1040         {
1041                 .type =       SOCK_DGRAM,
1042                 .protocol =   IPPROTO_UDP,
1043                 .prot =       &udp_prot,
1044                 .ops =        &inet_dgram_ops,
1045                 .no_check =   UDP_CSUM_DEFAULT,
1046                 .flags =      INET_PROTOSW_PERMANENT,
1047        },
1048 
1049        {
1050                 .type =       SOCK_DGRAM,
1051                 .protocol =   IPPROTO_ICMP,
1052                 .prot =       &ping_prot,
1053                 .ops =        &inet_dgram_ops,
1054                 .no_check =   UDP_CSUM_DEFAULT,
1055                 .flags =      INET_PROTOSW_REUSE,
1056        },
1057 
1058        {
1059                .type =       SOCK_RAW,
1060                .protocol =   IPPROTO_IP,        /* wild card */
1061                .prot =       &raw_prot,
1062                .ops =        &inet_sockraw_ops,
1063                .no_check =   UDP_CSUM_DEFAULT,
1064                .flags =      INET_PROTOSW_REUSE,
1065        }
1066 };
```
### where is it used
``` c
1670 static int __init inet_init(void)
1671 {
...
1725         /* Register the socket-side information for inet_create. */
1726         for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
1727                 INIT_LIST_HEAD(r);
1728 
1729         for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
1730                 inet_register_protosw(q);
...
```
## `struct proto`  vs `struct proto_ops` 
### `struct proto` and `tcp_prot`
``` c
 889 /* Networking protocol blocks we attach to sockets.
 890  * socket layer -> transport layer interface
 891  * transport -> network interface is defined by struct inet_proto
 892  */
 893 struct proto {
 894         void                    (*close)(struct sock *sk,
 895                                         long timeout);
 896         int                     (*connect)(struct sock *sk,
 897                                         struct sockaddr *uaddr,
 898                                         int addr_len);
 899         int                     (*disconnect)(struct sock *sk, int flags);
 900 
 901         struct sock *           (*accept)(struct sock *sk, int flags, int *err);
 902 
 903         int                     (*ioctl)(struct sock *sk, int cmd,
 904                                          unsigned long arg);
 905         int                     (*init)(struct sock *sk);
 906         void                    (*destroy)(struct sock *sk);
 907         void                    (*shutdown)(struct sock *sk, int how);
 908         int                     (*setsockopt)(struct sock *sk, int level,
 909                                         int optname, char __user *optval,
 910                                         unsigned int optlen);
 911         int                     (*getsockopt)(struct sock *sk, int level,
 912                                         int optname, char __user *optval,
 913                                         int __user *option);
 914 #ifdef CONFIG_COMPAT
 915         int                     (*compat_setsockopt)(struct sock *sk,
 916                                         int level,
 917                                         int optname, char __user *optval,
 918                                         unsigned int optlen);
 919         int                     (*compat_getsockopt)(struct sock *sk,
 920                                         int level,
 921                                         int optname, char __user *optval,
 922                                         int __user *option);
 923         int                     (*compat_ioctl)(struct sock *sk,
 924                                         unsigned int cmd, unsigned long arg);
 925 #endif
 926         int                     (*sendmsg)(struct kiocb *iocb, struct sock *sk,
 927                                            struct msghdr *msg, size_t len);
 928         int                     (*recvmsg)(struct kiocb *iocb, struct sock *sk,
 929                                            struct msghdr *msg,
 930                                            size_t len, int noblock, int flags,
 931                                            int *addr_len);
 932         int                     (*sendpage)(struct sock *sk, struct page *page,
 933                                         int offset, size_t size, int flags);
 934         int                     (*bind)(struct sock *sk,
 935                                         struct sockaddr *uaddr, int addr_len);
 936 
 937         int                     (*backlog_rcv) (struct sock *sk,
 938                                                 struct sk_buff *skb);
 939 
 940         void            (*release_cb)(struct sock *sk);
 941         void            (*mtu_reduced)(struct sock *sk);
 942 
 943         /* Keeping track of sk's, looking them up, and port selection methods. */
 944         void                    (*hash)(struct sock *sk);
 945         void                    (*unhash)(struct sock *sk);
 946         void                    (*rehash)(struct sock *sk);
 947         int                     (*get_port)(struct sock *sk, unsigned short snum);
 948         void                    (*clear_sk)(struct sock *sk, int size);
 949 
 950         /* Keeping track of sockets in use */
 951 #ifdef CONFIG_PROC_FS
 952         unsigned int            inuse_idx;
 953 #endif
 954 
 955         bool                    (*stream_memory_free)(const struct sock *sk);
 956         /* Memory pressure */
 957         void                    (*enter_memory_pressure)(struct sock *sk);
 958         atomic_long_t           *memory_allocated;      /* Current allocated memory. */
 959         struct percpu_counter   *sockets_allocated;     /* Current number of sockets. */
 960         /*
 961          * Pressure flag: try to collapse.
 962          * Technical note: it is used by multiple contexts non atomically.
 963          * All the __sk_mem_schedule() is of this nature: accounting
 964          * is strict, actions are advisory and have some latency.
 965          */
 966         int                     *memory_pressure;
 967         long                    *sysctl_mem;
 968         int                     *sysctl_wmem;
 969         int                     *sysctl_rmem;
 970         int                     max_header;
 971         bool                    no_autobind;
 972 
 973         struct kmem_cache       *slab;
 974         unsigned int            obj_size;
 975         int                     slab_flags;
 976 
 977         struct percpu_counter   *orphan_count;
 978 
 979         struct request_sock_ops *rsk_prot;
 980         struct timewait_sock_ops *twsk_prot;
 981 
 982         union {
 983                 struct inet_hashinfo    *hashinfo;
 984                 struct udp_table        *udp_table;
 985                 struct raw_hashinfo     *raw_hash;
 986         } h;
 987 
 988         struct module           *owner;
 989 
 990         char                    name[32];
 991 
 992         struct list_head        node;
 993 #ifdef SOCK_REFCNT_DEBUG
 994         atomic_t                socks;
 995 #endif
 996 #ifdef CONFIG_MEMCG_KMEM
 997         /*
 998          * cgroup specific init/deinit functions. Called once for all
 999          * protocols that implement it, from cgroups populate function.
1000          * This function has to setup any files the protocol want to
1001          * appear in the kmem cgroup filesystem.
1002          */
1003         int                     (*init_cgroup)(struct mem_cgroup *memcg,
1004                                                struct cgroup_subsys *ss);
1005         void                    (*destroy_cgroup)(struct mem_cgroup *memcg);
1006         struct cg_proto         *(*proto_cgroup)(struct mem_cgroup *memcg);
1007 #endif
1008 };
```

```c
2781 struct proto tcp_prot = {
2782         .name                   = "TCP",
2783         .owner                  = THIS_MODULE,
2784         .close                  = tcp_close,
2785         .connect                = tcp_v4_connect,
2786         .disconnect             = tcp_disconnect,
2787         .accept                 = inet_csk_accept,
2788         .ioctl                  = tcp_ioctl,
2789         .init                   = tcp_v4_init_sock,
2790         .destroy                = tcp_v4_destroy_sock,
2791         .shutdown               = tcp_shutdown,
2792         .setsockopt             = tcp_setsockopt,
2793         .getsockopt             = tcp_getsockopt,
2794         .recvmsg                = tcp_recvmsg,
2795         .sendmsg                = tcp_sendmsg,
2796         .sendpage               = tcp_sendpage,
2797         .backlog_rcv            = tcp_v4_do_rcv,
2798         .release_cb             = tcp_release_cb,
2799         .mtu_reduced            = tcp_v4_mtu_reduced,
2800         .hash                   = inet_hash,
2801         .unhash                 = inet_unhash,
2802         .get_port               = inet_csk_get_port,
2803         .enter_memory_pressure  = tcp_enter_memory_pressure,
2804         .stream_memory_free     = tcp_stream_memory_free,
2805         .sockets_allocated      = &tcp_sockets_allocated,
2806         .orphan_count           = &tcp_orphan_count,
2807         .memory_allocated       = &tcp_memory_allocated,
2808         .memory_pressure        = &tcp_memory_pressure,
2809         .sysctl_wmem            = sysctl_tcp_wmem,
2810         .sysctl_rmem            = sysctl_tcp_rmem,
2811         .max_header             = MAX_TCP_HEADER,
2812         .obj_size               = sizeof(struct tcp_sock),
2813         .slab_flags             = SLAB_DESTROY_BY_RCU,
2814         .twsk_prot              = &tcp_timewait_sock_ops,
2815         .rsk_prot               = &tcp_request_sock_ops,
2816         .h.hashinfo             = &tcp_hashinfo,
2817         .no_autobind            = true,
2818 #ifdef CONFIG_COMPAT
2819         .compat_setsockopt      = compat_tcp_setsockopt,
2820         .compat_getsockopt      = compat_tcp_getsockopt,
2821 #endif
2822 #ifdef CONFIG_MEMCG_KMEM
2823         .init_cgroup            = tcp_init_cgroup,
2824         .destroy_cgroup         = tcp_destroy_cgroup,
2825         .proto_cgroup           = tcp_proto_cgroup,
2826 #endif
2827 };
2828 EXPORT_SYMBOL(tcp_prot);
```

### `struct proto_ops` and `inet_stream_ops`
```c
127 struct proto_ops {
128         int             family;
129         struct module   *owner;
130         int             (*release)   (struct socket *sock);
131         int             (*bind)      (struct socket *sock,
132                                       struct sockaddr *myaddr,
133                                       int sockaddr_len);
134         int             (*connect)   (struct socket *sock,
135                                       struct sockaddr *vaddr,
136                                       int sockaddr_len, int flags);
137         int             (*socketpair)(struct socket *sock1,
138                                       struct socket *sock2);
139         int             (*accept)    (struct socket *sock,
140                                       struct socket *newsock, int flags);
141         int             (*getname)   (struct socket *sock,
142                                       struct sockaddr *addr,
143                                       int *sockaddr_len, int peer);
144         unsigned int    (*poll)      (struct file *file, struct socket *sock,
145                                       struct poll_table_struct *wait);
146         int             (*ioctl)     (struct socket *sock, unsigned int cmd,
147                                       unsigned long arg);
148 #ifdef CONFIG_COMPAT
149         int             (*compat_ioctl) (struct socket *sock, unsigned int cmd,
150                                       unsigned long arg);
151 #endif
152         int             (*listen)    (struct socket *sock, int len);
153         int             (*shutdown)  (struct socket *sock, int flags);
154         int             (*setsockopt)(struct socket *sock, int level,
155                                       int optname, char __user *optval, unsigned int optlen);
156         int             (*getsockopt)(struct socket *sock, int level,
157                                       int optname, char __user *optval, int __user *optlen);
158 #ifdef CONFIG_COMPAT
159         int             (*compat_setsockopt)(struct socket *sock, int level,
160                                       int optname, char __user *optval, unsigned int optlen);
161         int             (*compat_getsockopt)(struct socket *sock, int level,
162                                       int optname, char __user *optval, int __user *optlen);
163 #endif
164         int             (*sendmsg)   (struct kiocb *iocb, struct socket *sock,
165                                       struct msghdr *m, size_t total_len);
166         int             (*recvmsg)   (struct kiocb *iocb, struct socket *sock,
167                                       struct msghdr *m, size_t total_len,
168                                       int flags);
169         int             (*mmap)      (struct file *file, struct socket *sock,
170                                       struct vm_area_struct * vma);
171         ssize_t         (*sendpage)  (struct socket *sock, struct page *page,
172                                       int offset, size_t size, int flags);
173         ssize_t         (*splice_read)(struct socket *sock,  loff_t *ppos,
174                                        struct pipe_inode_info *pipe, size_t len, unsigned int flags);
175         void            (*set_peek_off)(struct sock *sk, int val);
176 };
```

```c
 934 const struct proto_ops inet_stream_ops = {
 935         .family            = PF_INET,
 936         .owner             = THIS_MODULE,
 937         .release           = inet_release,
 938         .bind              = inet_bind,
 939         .connect           = inet_stream_connect,
 940         .socketpair        = sock_no_socketpair,
 941         .accept            = inet_accept,
 942         .getname           = inet_getname,
 943         .poll              = tcp_poll,
 944         .ioctl             = inet_ioctl,
 945         .listen            = inet_listen,
 946         .shutdown          = inet_shutdown, 
 947         .setsockopt        = sock_common_setsockopt,
 948         .getsockopt        = sock_common_getsockopt,
 949         .sendmsg           = inet_sendmsg,
 950         .recvmsg           = inet_recvmsg,
 951         .mmap              = sock_no_mmap,
 952         .sendpage          = inet_sendpage,
 953         .splice_read       = tcp_splice_read,
 954 #ifdef CONFIG_COMPAT                  
 955         .compat_setsockopt = compat_sock_common_setsockopt, 
 956         .compat_getsockopt = compat_sock_common_getsockopt,
 957         .compat_ioctl      = inet_compat_ioctl,
 958 #endif
 959 };
 960 EXPORT_SYMBOL(inet_stream_ops);
```

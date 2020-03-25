---
layout: post
title: "the inheriting of linux sock type"
date: 2013-11-22 17:50
comments: true
categories: [socket]
tags: [socket]
---

### Data Structure
Every `xsock` has parent sock as its first filed.

<!-- more -->

1. `sock_common`
```c
157 struct sock_common {
...
}
```
2. `sock`
```c
 288 struct sock {
 289         /*
 290          * Now struct inet_timewait_sock also uses sock_common, so please just
 291          * don't add nothing before this first member (__sk_common) --acme
 292          */
 293         struct sock_common      __sk_common;
...
```
3. `inet_sock`
```c
37 struct inet_sock {
138         /* sk and pinet6 has to be the first two members of inet_sock */
139         struct sock             sk;
...
```
4. `inet_connection_sock`
```c
 87 struct inet_connection_sock {
 88         /* inet_sock has to be the first member! */
 89         struct inet_sock          icsk_inet;
...
```
5. `tcp_sock`
```c
133 struct tcp_sock {
134         /* inet_connection_sock has to be the first member of tcp_sock */
135         struct inet_connection_sock     inet_conn;
...
```


#### 转换函数：

##### sk to `inet_sock`
```
213 static inline struct inet_sock *inet_sk(const struct sock *sk)
214 {
215         return (struct inet_sock *)sk;
216 }
```

##### sk to `inet_connection_sock`
```
145 static inline struct inet_connection_sock *inet_csk(const struct sock *sk)
146 {
147         return (struct inet_connection_sock *)sk;
148 }
```

##### sk to `tcp_sk`
```
351 static inline struct tcp_sock *tcp_sk(const struct sock *sk)
352 {
353         return (struct tcp_sock *)sk;
354 }
```
![Inherit](/images/sock/tcpsock.jpg)

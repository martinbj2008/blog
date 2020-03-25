---
layout: post
title: "IPv4 route study part 1: basic fun"
date: 2013-08-26 11:05
comments: true
categories: [route]
tags: [IPv4, route,lc-trie]
---

In order to understand kernel route LC-Trie,
summary the basic functions here.

<!-- more -->

## NODE vs LEAF
`node` and `leaf` have two same head elements

1. `unsigned long parent`
2. `t_key key`

We can treat `leaf` as `node`+`leaf extention`
```c
	struct leaf {
		struct node;
	....
}
```

### node
```c
  98 struct node {
  99         unsigned long parent;
 100         t_key key;
 101 };
```
### leaf
```c
 103 struct leaf {
 104         unsigned long parent;
 105         t_key key;
 106         struct hlist_head list;
 107         struct rcu_head rcu;
 108 };
```

## PARENT

It is a common method, hide some bit flags into the low bits of a pointer.
because the low bits always is zero(becauase of CACHE).

###the lowest 1 bit of `parent` is used to different `node` or `leaf`.

```c
  90 #define T_TNODE 0
  91 #define T_LEAF  1
  92 #define NODE_TYPE_MASK  0x1UL
  93 #define NODE_TYPE(node) ((node)->parent & NODE_TYPE_MASK)
  94 
  95 #define IS_TNODE(n) (!(n->parent & T_LEAF))
  96 #define IS_LEAF(n) (n->parent & T_LEAF)
```

```c
 180 static inline struct tnode *node_parent(struct node *node)
 181 {
 182         return (struct tnode *)(node->parent & ~NODE_TYPE_MASK);
 183 }
 184 
 185 static inline struct tnode *node_parent_rcu(struct node *node)
 186 {
 187         struct tnode *ret = node_parent(node);
 188 
 189          rcu_dereference_rtnl(ret);
 190 }
 191 
 192 /* Same as rcu_assign_pointer
 193  * but that macro() assumes that value is a pointer.
 194  */
 195 static inline void node_set_parent(struct node *node, struct tnode *ptr)
 196 {
 197         smp_wmb();
 198         node->parent = (unsigned long)ptr | NODE_TYPE(node);
 199 }
```

##BIT OPS
```c
 220 static inline t_key mask_pfx(t_key k, unsigned short l)
 221 {
 222         return (l == 0) ? 0 : k >> (KEYLENGTH-l) << (KEYLENGTH-l);
 223 }
```

### `tkey_extract_bits`

Get value of key's bits from the `offset` bit.
取 从第offset位开始的 bits位的值

![tkey_extract_bits](/images/fib_trie/tkey_extract_bits.png)

```c 
 225 static inline t_key tkey_extract_bits(t_key a, int offset, int bits)
 226 {
 227         if (offset < KEYLENGTH)
 228                 return ((t_key)(a << offset)) >> (KEYLENGTH - bits);
 229         else
 230                 return 0;
 231 }
```

### `tkey_equals`
`tkey a` is equal with `tkey b`
```c 
 233 static inline int tkey_equals(t_key a, t_key b)
 234 {
 235         return a == b;
 236 }
```
### `tkey_sub_equals`
simlar with `tkey_equals`, while it only compare the `bits` bits from the 
`offset`st bit.

```c 
 238 static inline int tkey_sub_equals(t_key a, int offset, int bits, t_key b)
 239 {
 240         if (bits == 0 || offset >= KEYLENGTH)
 241                 return 1;
 242         bits = bits > KEYLENGTH ? KEYLENGTH : bits;
 243         return ((a ^ b) << offset) >> (KEYLENGTH - bits) == 0;
 244 }
```

### `tkey_mismatch`
find out the first different bit after the `offset`st bit
between `tkey a` and `tkey b`.

after the operation `a ^ b`, the first bit with `1' will 
be the first different bit.

![tkey_mismatch](/images/fib_trie/tkey_mismatch.png)

```c 
 246 static inline int tkey_mismatch(t_key a, int offset, t_key b)
 247 {
 248         t_key diff = a ^ b;
 249         int i = offset;
 250 
 251         if (!diff)
 252                 return 0;
 253         while ((diff << i) >> (KEYLENGTH-1) == 0)
 254                 i++;
 255         return i;
 256 }
```

### `mask_pfx`
保留`key`的前`l`位，并将`l`位之后的所有位清零。`l`相当于掩码长度。

```c
 238 static inline t_key mask_pfx(t_key k, unsigned int l)
 239 { 
 240         return (l == 0) ? 0 : k >> (KEYLENGTH-l) << (KEYLENGTH-l); 
 241 } 
```

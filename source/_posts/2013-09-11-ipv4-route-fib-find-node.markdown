---
layout: post
title: "IPv4 route fib find node"
date: 2013-09-11 17:15
comments: true
categories: [route]
tags: [kernel, IPv4, fib]
---

###summary
遍历路由树，查找是否存在`key`这条路由。
如果有则返回对应的节点 `struct leaf *`，
否则返回NULL。


#### 中间节点 和 叶子节点
路由树（fib lc tree)是个多叉树,树的每个节点是`struct rt_trie_node`。
每个节点的分叉个数(`X`)是可变的， 由 `struct rt_trie_node`里的`bits`决定。
  `X` = 2 的`bits`次幂次方。

叶子节点的首个子类型也是`struct rt_trie_node`，所以可以看成是其派生。

#### 树的遍历
1. 如果当前节点为空， 结束循环。
2. 如果当前节点不是`T_TNODE`，结束循环。
    即:当前节点为 叶子节点`T_LEAF`.

3. 比较当前节点`tn`和参数`key`的`未比较过的前缀`是否匹配。
    `tkey_sub_equals(tn->key, x, y, key))`

   `未比较过的前缀`指`[x-y)`这个区间的位，包括`x`,不包括`y`。
   `x` = 父节点的`pos` + 父节点的`bits` （父节点为空时，x取0）
   `y` = 当前节点的`pos` + 父节点的`pos`
   这里，具体取哪些位石油父节点和当前节点刚提决定的。

3. 如果相等，则结束循环，判断当前节点是否真正匹配。
4. 如果不相等，则跳到第`N`个一个孩子节点。 
   `N = tkey_extract_bits(key, tn->pos, tn->bits)`
   `N`是取从参数`key`的第`tn->pos`位开始, 共`tn->bits`位转化出来的值。
   注意，此处是取`key`的某一位或者几位的值，而具体怎么取值是有当前节点的(pos,bits)决定的。
   

#### 完整判断
当前节点`n`是叶子节点，并且`key`完全相同
 `n != NULL && IS_LEAF(n) && tkey_equals(key, n->key))`


<!-- more -->

```c
 954 static struct leaf *
 955 fib_find_node(struct trie *t, u32 key)
 956 {
 957         int pos;
 958         struct tnode *tn;
 959         struct rt_trie_node *n;
 960 
 961         pos = 0;
 962         n = rcu_dereference_rtnl(t->trie);
 963 
 964         while (n != NULL &&  NODE_TYPE(n) == T_TNODE) {
 965                 tn = (struct tnode *) n;
 966 
 967                 check_tnode(tn);
 968 
 969                 if (tkey_sub_equals(tn->key, pos, tn->pos-pos, key)) {
 970                         pos = tn->pos + tn->bits;
 971                         n = tnode_get_child_rcu(tn,
 972                                                 tkey_extract_bits(key,
 973                                                                   tn->pos,
 974                                                                   tn->bits));
 975                 } else
 976                         break;
 977         }
 978         /* Case we have found a leaf. Compare prefixes */
 979 
 980         if (n != NULL && IS_LEAF(n) && tkey_equals(key, n->key))
 981                 return (struct leaf *)n;
 982 
 983         return NULL;
 984 }
```

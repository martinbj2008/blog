---
layout: post
title: "IPv4 route fib insert node"
date: 2013-09-12 09:48
comments: true
categories: [route]
tags: [kernel, IPv4, fib]
---

##steps

step 1. 循环`fib_trie`, 直到当前节点为空或者是叶子节点时，停止。
	1.a
	比较当前节点的`key`跟待插入的`key`的前pos位,
	如果不相等，循环结束。如果相等执行1.b
       （作为改进，加上父节点已经比较了前posx位， 那么只需比较
	(posx，pos)这区间的位即可。 注意posx有可能与pos相等。
	1.b
	记录已经比较的位数(tn->pos + tn->bits)
	取当前节点的一个孩子节点，
	`tkey_extract_bits(key, tn->pos, tn->bits)`
	继续执行。

step 2. 循环结束，当前节点`n`有可能为空，也有可能是个叶子节点,或者中间节点(1.a).
```c
	if (n == NULL)
		if (fib_trie == NULL)
			shit! 树的第一个节点。
		else
			当前节点是父节点的一个空孩子节点。
			以为树不是空的，所以父节点肯定存在。
			见 case2
	else n is a leaf 
		if (tkey_equals(key, n->key)) //'key'值相同
			case 1:
			路由前缀相同（可能掩码不一样)
		else
			case 3b.
	else
		//assert n is a internal node
		并且，当前节点的key的pos位跟待插入的key的前pos位不相等。
		见case3a

```
<!-- more -->

When insert a route to the `fib_trie`, it 
can be divided into 3 cases.

###case 1
We are lucky enough, there are matched leaf.
for example, insert a subnet route to a exist route.

![case 1](/images/fib_trie/insert_node_case1.jpg)

###case 2
There is a empty child slot in the matched node,
we just need insert a leaf to the `fib_trie`.

![case 2](/images/fib_trie/insert_node_case2.jpg)

###case 3
There is a umatched node, we need insert a internal node
to the `fib_trie`, and insert a leaf as one of the internal
node.

####case 3a
The unmatched node is a node(`sub-fib-trie`).

![case 3a](/images/fib_trie/insert_node_case3a.jpg)

####case 3b
The unmatched node is a leaf.
![case 3b](/images/fib_trie/insert_node_case3b.jpg)

### source `fib_insert_node`
```c
1021 /* only used from updater-side */
1022 
1023 static struct list_head *fib_insert_node(struct trie *t, u32 key, int plen)
1024 {
1025         int pos, newpos;
1026         struct tnode *tp = NULL, *tn = NULL;
1027         struct rt_trie_node *n;
1028         struct leaf *l;
1029         int missbit;
1030         struct list_head *fa_head = NULL;
1031         struct leaf_info *li;
1032         t_key cindex;
1033 
1034         pos = 0;
1035         n = rtnl_dereference(t->trie);
1036 
1037         /* If we point to NULL, stop. Either the tree is empty and we should
1038          * just put a new leaf in if, or we have reached an empty child slot,
1039          * and we should just put our new leaf in that.
1040          * If we point to a T_TNODE, check if it matches our key. Note that
1041          * a T_TNODE might be skipping any number of bits - its 'pos' need
1042          * not be the parent's 'pos'+'bits'!
1043          *
1044          * If it does match the current key, get pos/bits from it, extract
1045          * the index from our key, push the T_TNODE and walk the tree.
1046          *
1047          * If it doesn't, we have to replace it with a new T_TNODE.
1048          *
1049          * If we point to a T_LEAF, it might or might not have the same key
1050          * as we do. If it does, just change the value, update the T_LEAF's
1051          * value, and return it.
1052          * If it doesn't, we need to replace it with a T_TNODE.
1053          */
1054 
1055         while (n != NULL &&  NODE_TYPE(n) == T_TNODE) {
1056                 tn = (struct tnode *) n;
1057 
1058                 check_tnode(tn);
1059 
1060                 if (tkey_sub_equals(tn->key, pos, tn->pos-pos, key)) {
1061                         tp = tn;
1062                         pos = tn->pos + tn->bits;
1063                         n = tnode_get_child(tn,
1064                                             tkey_extract_bits(key,
1065                                                               tn->pos,
1066                                                               tn->bits));
1067 
1068                         BUG_ON(n && node_parent(n) != tn);
1069                 } else
1070                         break;
1071         }
1072 
1073         /*
1074          * n  ----> NULL, LEAF or TNODE
1075          *
1076          * tp is n's (parent) ----> NULL or TNODE
1077          */
1078 
1079         BUG_ON(tp && IS_LEAF(tp));
1080 
1081         /* Case 1: n is a leaf. Compare prefixes */
1082 
1083         if (n != NULL && IS_LEAF(n) && tkey_equals(key, n->key)) {
1084                 l = (struct leaf *) n;
1085                 li = leaf_info_new(plen);
1086 
1087                 if (!li)
1088                         return NULL;
1089 
1090                 fa_head = &li->falh;
1091                 insert_leaf_info(&l->list, li);
1092                 goto done;
1093         }
1094         l = leaf_new();
1095 
1096         if (!l)
1097                 return NULL;
1098 
1099         l->key = key;
1100         li = leaf_info_new(plen);
1101 
1102         if (!li) {
1103                 free_leaf(l);
1104                 return NULL;
1105         }
1106 
1107         fa_head = &li->falh;
1108         insert_leaf_info(&l->list, li);
1109 
1110         if (t->trie && n == NULL) {
1111                 /* Case 2: n is NULL, and will just insert a new leaf */
1112 
1113                 node_set_parent((struct rt_trie_node *)l, tp);
1114 
1115                 cindex = tkey_extract_bits(key, tp->pos, tp->bits);
1116                 put_child(tp, cindex, (struct rt_trie_node *)l);
1117         } else {
1118                 /* Case 3: n is a LEAF or a TNODE and the key doesn't match. */
1119                 /*
1120                  *  Add a new tnode here
1121                  *  first tnode need some special handling
1122                  */
1123 
1124                 if (tp)
1125                         pos = tp->pos+tp->bits;
1126                 else
1127                         pos = 0;
1128 
1129                 if (n) {
1130                         newpos = tkey_mismatch(key, pos, n->key); /* newpos < pos, 因为1083行，见注释1 */
1131                         tn = tnode_new(n->key, newpos, 1);
1132                 } else {
1133                         newpos = 0;
1134                         tn = tnode_new(key, newpos, 1); /* First tnode */
1135                 }
1136 
1137                 if (!tn) {
1138                         free_leaf_info(li);
1139                         free_leaf(l);
1140                         return NULL;
1141                 }
1142 
1143                 node_set_parent((struct rt_trie_node *)tn, tp);
1144 
1145                 missbit = tkey_extract_bits(key, newpos, 1);
1146                 put_child(tn, missbit, (struct rt_trie_node *)l);
1147                 put_child(tn, 1-missbit, n);
1148 
1149                 if (tp) {
1150                         cindex = tkey_extract_bits(key, tp->pos, tp->bits);
1151                         put_child(tp, cindex, (struct rt_trie_node *)tn);
1152                 } else {
1153                         rcu_assign_pointer(t->trie, (struct rt_trie_node *)tn);
1154                         tp = tn;
1155                 }
1156         }
1157 
1158         if (tp && tp->pos + tp->bits > 32)
1159                 pr_warn("fib_trie tp=%p pos=%d, bits=%d, key=%0x plen=%d\n",
1160                         tp, tp->pos, tp->bits, key, plen);
1161 
1162         /* Rebalance the trie */
1163 
1164         trie_rebalance(t, tp);
1165 done:
1166         return fa_head;
1167 }
1168 
```

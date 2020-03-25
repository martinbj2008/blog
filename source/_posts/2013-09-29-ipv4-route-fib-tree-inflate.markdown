---
layout: post
title: "IPv4 route fib trie inflate"
date: 2013-09-29 17:16
comments: true
categories: [route]
ctags: [kernel, ipv4, route, trie]
---

##summary
`inflate` duplicate a new node whose child array is double of orignal node,
and put original node's child into the new node.

<!-- more -->

```c
tn's bits = 2 * oldnode->bits

`struct tnode *oldnode` is the orignal node.
`struct tnode *tn` is the new malloc node.
```

Just like the comment in the source, child nodes are divided into 3 kind nodes:

###case 1. child node is a leaf or an internal node with skipped bits.
	节点不变，直接挂到新的节点下面，作为第2*i或者2*i+1个孩子节点。

![case 1](/images/fib_trie/inflate.case1.jpg)

###case 2. an internal node with two children.
	释放节点，并将其左右两个孩子挂到新的节点下面，依次作第2*i和2*i+1个孩子节点。

![case 2](/images/fib_trie/inflate.case2.jpg)

###case 3. an internal node with more than two children.
        释放节点，该节点的孩子节点从中间一分为二， 并分别被复制到两个新的节点（
	left和right）。这两个新节点被依次作为新节点的第2*i和2*i+1孩子节点。
	left和right节点要重新经过`resize`函数处理，然后再挂到新节点上。

![case 3](/images/fib_trie/inflate.case3.jpg)


### NOTE:
1. 预分配内存：
	    为了防止中间过程内存不足，在函数一开始，就先为第三类节点分配足够的内存(只有第三类情况需要新建节点)。
	    等真正处理的时候，只是先把left和right节点从新节点上取下来，待把中间节点(internal node)的所有子孩子重新分配后，再重新把left和right节点挂到新节点下。

2. skipped bits:
	a node's `pos` is bigger than parent nodes's `(pos+bits)`.
<!-- more -->

### how to divid child nodes
```c
if (a leaf node) then
    belong case1.
else //assert: it must be a internal node).
     if (it has skipped bits) then
        belong case1
     else //assert: pos= parent's(pos+bits)
        if (two child nodes)
            belong case2
        else (more than two child nodes)
            belong case3
            (4,8,16... child nodes)
```

### `inflate`
```c
   702	static struct tnode *inflate(struct trie *t, struct tnode *tn)
   703	{
   704		struct tnode *oldtnode = tn;
   705		int olen = tnode_child_length(tn);
   706		int i;
   707	
   708		pr_debug("In inflate\n");
   709		/*扩展一个新的节点，其孩子节点的个数加倍*/
   710		tn = tnode_new(oldtnode->key, oldtnode->pos, oldtnode->bits + 1);
   711	
   712		if (!tn)
   713			return ERR_PTR(-ENOMEM);
   714	
   715		/*
   716		 * Preallocate and store tnodes before the actual work so we
   717		 * don't get into an inconsistent state if memory allocation
   718		 * fails. In case of failure we return the oldnode and  inflate
   719		 * of tnode is ignored.
   720		 */
   721	
   722		for (i = 0; i < olen; i++) {
   723			struct tnode *inode;
   724	
   725			inode = (struct tnode *) tnode_get_child(oldtnode, i);
   726			if (inode &&
   727			    IS_TNODE(inode) &&
   728			    inode->pos == oldtnode->pos + oldtnode->bits &&
   729			    inode->bits > 1) {
   730				struct tnode *left, *right;
   731				t_key m = ~0U << (KEYLENGTH - 1) >> inode->pos;
   732	
   733				left = tnode_new(inode->key&(~m), inode->pos + 1,
   734						 inode->bits - 1);
   735				if (!left)
   736					goto nomem;
   737	
   738				right = tnode_new(inode->key|m, inode->pos + 1,
   739						  inode->bits - 1);
   740	
   741				if (!right) {
   742					tnode_free(left);
   743					goto nomem;
   744				}
   745	
   746				put_child(tn, 2*i, (struct rt_trie_node *) left);
   747				put_child(tn, 2*i+1, (struct rt_trie_node *) right);
   748			}
   749		}
   750	
   751		for (i = 0; i < olen; i++) {
   752			struct tnode *inode;
   753			struct rt_trie_node *node = tnode_get_child(oldtnode, i);
   754			struct tnode *left, *right;
   755			int size, j;
   756	
   757			/* An empty child */
   758			if (node == NULL)
   759				continue;
   760	
   761			/* A leaf or an internal node with skipped bits */
   762	
   763			if (IS_LEAF(node) || ((struct tnode *) node)->pos >
   764			   tn->pos + tn->bits - 1) {
   765				put_child(tn,
   766					tkey_extract_bits(node->key, oldtnode->pos, oldtnode->bits + 1),
   767					node);
   768				continue;
   769			}
   770	
   771			/* An internal node with two children */
   772			inode = (struct tnode *) node;
   773	
   774			if (inode->bits == 1) {
   775				put_child(tn, 2*i, rtnl_dereference(inode->child[0]));
   776				put_child(tn, 2*i+1, rtnl_dereference(inode->child[1]));
   777	
   778				tnode_free_safe(inode);
   779				continue;
   780			}
   781	
   782			/* An internal node with more than two children */
   783	
   784			/* We will replace this node 'inode' with two new
   785			 * ones, 'left' and 'right', each with half of the
   786			 * original children. The two new nodes will have
   787			 * a position one bit further down the key and this
   788			 * means that the "significant" part of their keys
   789			 * (see the discussion near the top of this file)
   790			 * will differ by one bit, which will be "0" in
   791			 * left's key and "1" in right's key. Since we are
   792			 * moving the key position by one step, the bit that
   793			 * we are moving away from - the bit at position
   794			 * (inode->pos) - is the one that will differ between
   795			 * left and right. So... we synthesize that bit in the
   796			 * two  new keys.
   797			 * The mask 'm' below will be a single "one" bit at
   798			 * the position (inode->pos)
   799			 */
   800	
   801			/* Use the old key, but set the new significant
   802			 *   bit to zero.
   803			 */
   804	
   805			left = (struct tnode *) tnode_get_child(tn, 2*i);
   806			put_child(tn, 2*i, NULL);
   807	
   808			BUG_ON(!left);
   809	
   810			right = (struct tnode *) tnode_get_child(tn, 2*i+1);
   811			put_child(tn, 2*i+1, NULL);
   812	
   813			BUG_ON(!right);
   814	
   815			size = tnode_child_length(left);
   816			for (j = 0; j < size; j++) {
   817				put_child(left, j, rtnl_dereference(inode->child[j]));
   818				put_child(right, j, rtnl_dereference(inode->child[j + size]));
   819			}
   820			put_child(tn, 2*i, resize(t, left));
   821			put_child(tn, 2*i+1, resize(t, right));
   822	
   823			tnode_free_safe(inode);
   824		}
   825		tnode_free_safe(oldtnode);
   826		return tn;
   827	nomem:
   828		tnode_clean_free(tn);
   829		return ERR_PTR(-ENOMEM);
   830	}
```

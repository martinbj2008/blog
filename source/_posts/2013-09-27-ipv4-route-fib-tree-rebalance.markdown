---
layout: post
title: "IPv4 route fib tree rebalance"
date: 2013-09-27 16:36
comments: true
categories: [route]
tags: [ipv4, route, kernel]
---

### call trace
以插入一条新的路由为例。

```c
> fib_insert_node
> > trie_rebalance
> > > while loop
> > > > resize
> > > > tnode_put_child_reorg
> > > > tnode_free_flush
```

###`trie_rebalance`

```
for_each_node(from current node tn to  fib_trie root)
	call resize()
	tnode_put_child_reorg

从当前节点开始一直到根节点，以当前节点作为一个子树，
反复调用resize, 并通过tnode_put_child_reorg
更新当前节点的父节点的统计信息。

注： resize可能会更改子树的根节点！
```

<!-- more -->

```c
 986 static void trie_rebalance(struct trie *t, struct tnode *tn)
 987 { 
 988         int wasfull;               
 989         t_key cindex, key;         
 990         struct tnode *tp;          
 991   
 992         key = tn->key;             
 993   
 994         while (tn != NULL && (tp = node_parent((struct rt_trie_node *)tn)) != NULL) {
 995                 cindex = tkey_extract_bits(key, tp->pos, tp->bits);
 996                 wasfull = tnode_full(tp, tnode_get_child(tp, cindex));
 997                 tn = (struct tnode *)resize(t, tn);      
 998   
 999                 tnode_put_child_reorg(tp, cindex,        
1000                                       (struct rt_trie_node *)tn, wasfull);     
1001   
1002                 tp = node_parent((struct rt_trie_node *) tn); 
1003                 if (!tp)           
1004                         rcu_assign_pointer(t->trie, (struct rt_trie_node *)tn);
1005   
1006                 tnode_free_flush();
1007                 if (!tp)           
1008                         break;     
1009                 tn = tp;           
1010         }
1011   
1012         /* Handle last (top) tnode */
1013         if (IS_TNODE(tn))          
1014                 tn = (struct tnode *)resize(t, tn);      
1015   
1016         rcu_assign_pointer(t->trie, (struct rt_trie_node *)tn);
1017         tnode_free_flush();        
1018 } 
```

### rebalance的核心`resize`函数

#### `inflate`和`halve`
rebalancede核心是resize函数。在该函数里有两种操作`inflate`和`halve`，他们是两种相反的操作。

1. `inflate`是将当前节点的孩子指针数组的元素个数增加一倍， 同时将尽量将各个子孩子的孩子，直接挂接到新的孩子指针数组上。
	最终达到减少树的深度。所以要像执行`inflate`操作，孙子节点必须足够多。

2. `halve`将当前节点的孩子指针数组的元素个数减半。
	如果第2i和2i+1个孩子都为空，则压缩后的第i个孩子也为空，
	如果第2i和2i+1个孩子只有一个为空，则将不非空的孩子作为压缩后的第i个孩子。
	如果第2i和2i+1个孩子都不为空，则插入一个新的中间节点。并将第2i和2i+1个孩子分别作为新的中间节点的左右(第0和第1个)孩子。
![case 1](/images/fib_trie//halve.jpg)

#### `inflate`和`halve`的执行条件
英文注释说的很清楚了。见下文的翻译。


#### `resize`
```c
523 #define MAX_WORK 10
 524 static struct rt_trie_node *resize(struct trie *t, struct tnode *tn)
 525 {
 526         int i;
 527         struct tnode *old_tn;
 528         int inflate_threshold_use; 
 529         int halve_threshold_use;
 530         int max_work;
 531         
 532         if (!tn)
 533                 return NULL; 
 534   
 535         pr_debug("In tnode_resize %p inflate_threshold=%d threshold=%d\n",
 536                  tn, inflate_threshold, halve_threshold);
 537 
 538         /* No children */
 539         if (tn->empty_children == tnode_child_length(tn)) {
 540                 tnode_free_safe(tn);
 541                 return NULL;
 542         }
 543         /* One child */
 544         if (tn->empty_children == tnode_child_length(tn) - 1)
 545                 goto one_child;    
 546         /*
 547          * Double as long as the resulting node has a number of
 548          * nonempty nodes that are above the threshold.
 549          */
 550         
 551         /*                         
 552          * From "Implementing a dynamic compressed trie" by Stefan Nilsson of
 553          * the Helsinki University of Technology and Matti Tikkanen of Nokia
 554          * Telecommunications, page 6:
 555          * "A node is doubled if the ratio of non-empty children to all
 556          * children in the *doubled* node is at least 'high'."

		根据 "Implementing a dynamic compressed trie"的第6页
		"如果想要一个节点的， 其非空子孩子在所有孩子中的比例必须不少于 ‘high'"

 557          *
 558          * 'high' in this instance is the variable 'inflate_threshold'. It
 559          * is expressed as a percentage, so we multiply it with
 560          * tnode_child_length() and instead of multiplying by 2 (since the
 561          * child array will be doubled by inflate()) and multiplying
 562          * the left-hand side by 100 (to handle the percentage thing) we
 563          * multiply the left-hand side by 50.
 564          *
		'high'在这里有变量'inflate_threshold'表示。它有一个百分比的形式。
		100 *  tn_noempty_children/(tnode_child_length() *2) = 'inflate_threshold'  
		避免除法:
		100 * tn_noempty_children = 'inflate_threshold' * (tnode_child_length() *2)
		100和2 抵消： 
		50 * tn_noempty_children = 'inflate_threshold' * tnode_child_length()
		展开tn_noempty_children
		50 * (tnode_child_length(tn) - tn->empty_children) = 'inflate_threshold' * tnode_child_length() 

 565          * The left-hand side may look a bit weird: tnode_child_length(tn)
 566          * - tn->empty_children is of course the number of non-null children
 567          * in the current node. tn->full_children is the number of "full"
 568          * children, that is non-null tnodes with a skip value of 0.
 569          * All of those will be doubled in the resulting inflated tnode, so
 570          * we just count them one extra time here.
 571          *
		左边看起来有些怪异， 
		tnode_child_length(tn) - tn->empty_children 指的是当前节点的非空孩子节点个数。
		tn->full_children 是 'full'孩子节点的个数。即那些非空节点并且节点的
		skip值为0（即 节点的pos == 父节点的pos+父节点的bits）
		在最终膨胀的节点里，这些节点将会膨胀原来的两倍。

 572          * A clearer way to write this would be:
 573          *
 574          * to_be_doubled = tn->full_children;
 575          * not_to_be_doubled = tnode_child_length(tn) - tn->empty_children -
 576          *     tn->full_children;
 577          *
 578          * new_child_length = tnode_child_length(tn) * 2;
 579          *
		要膨胀加倍的节点 = tn->full_children;
		不需要膨胀的非空节点 = tnode_child_length(tn) - tn->empty_children - tn->full_children;
		新的孩子节点总数 =  tnode_child_length(tn) * 2;

 580          * new_fill_factor = 100 * (not_to_be_doubled + 2*to_be_doubled) /
 581          *      new_child_length;
 582          * if (new_fill_factor >= inflate_threshold)
 583          *
 584          * ...and so on, tho it would mess up the while () loop.
 585          *
		new_fill_factor = 100 * （不需要膨胀的非空节点 + 2 * 要膨胀加倍的节点) / 新的孩子节点总数
		if (new_fill_factor > inflate_threshold)
			就变得麻烦了， 我们需要一个while loop来处理。

 586          * anyway,
 587          * 100 * (not_to_be_doubled + 2*to_be_doubled) / new_child_length >=
 588          *      inflate_threshold
 589          *
		总之，
		100 * （不需要膨胀的非空节点 + 2 * 要膨胀加倍的节点) / 新的孩子节点总数 >= inflate_threshold
 
 590          * avoid a division:
 591          * 100 * (not_to_be_doubled + 2*to_be_doubled) >=
 592          *      inflate_threshold * new_child_length
 593          *

		避免除法
		100 * （不需要膨胀的非空节点 + 2 * 要膨胀加倍的节点) >= inflate_threshold * 新的孩子节点总数

 594          * expand not_to_be_doubled and to_be_doubled, and shorten:
 595          * 100 * (tnode_child_length(tn) - tn->empty_children +
 596          *    tn->full_children) >= inflate_threshold * new_child_length
 597          *
		展开 不需要膨胀的非空节点 与 要膨胀加倍的节点
		100 *（tnode_child_length(tn) - tn->empty_children - tn->full_children + 2 * tn->full_children)
			>= inflate_threshold * 新的孩子节点总数
		===>
		100 * (tnode_child_length(tn) - tn->empty_children + tn->full_children) >= inflate_threshold * 新的孩子节点总数

 598          * expand new_child_length:
 599          * 100 * (tnode_child_length(tn) - tn->empty_children +
 600          *    tn->full_children) >=
 601          *      inflate_threshold * tnode_child_length(tn) * 2
 602          *
		展开 新的孩子节点总数
		100 * (tnode_child_length(tn) - tn->empty_children + tn->full_children) >= inflate_threshold * tnode_child_length(tn) * 2;

 603          * shorten again:
 604          * 50 * (tn->full_children + tnode_child_length(tn) -
 605          *    tn->empty_children) >= inflate_threshold *
 606          *    tnode_child_length(tn)
 607          *
		再化简(2 被100 抵消):
                50 * (tnode_child_length(tn) - tn->empty_children + tn->full_children) >= inflate_threshold * tnode_child_length(tn);
		调整位置:
		50 * (tn->full_children + tnode_child_length(tn) - tn->empty_children)
			>= inflate_threshold * tnode_child_length(tn);
 608          */
 609 
 610         check_tnode(tn);
 611 
 612         /* Keep root node larger  */
 613 
 614         if (!node_parent((struct rt_trie_node *)tn)) {
 615                 inflate_threshold_use = inflate_threshold_root;
 616                 halve_threshold_use = halve_threshold_root;
 617         } else {
 618                 inflate_threshold_use = inflate_threshold;
 619                 halve_threshold_use = halve_threshold;
 620         }
 621 
 622         max_work = MAX_WORK;
 623         while ((tn->full_children > 0 &&  max_work-- &&
 624                 50 * (tn->full_children + tnode_child_length(tn)
 625                       - tn->empty_children)
 626                 >= inflate_threshold_use * tnode_child_length(tn))) {
 627 
 628                 old_tn = tn;
 629                 tn = inflate(t, tn);
 630 
 631                 if (IS_ERR(tn)) {
 632                         tn = old_tn;
 633 #ifdef CONFIG_IP_FIB_TRIE_STATS
 634                         t->stats.resize_node_skipped++;
 635 #endif
 636                         break;
 637                 }
 638         }
 639 
 640         check_tnode(tn);
 641 
 642         /* Return if at least one inflate is run */
 643         if (max_work != MAX_WORK)
 644                 return (struct rt_trie_node *) tn;
 645 
 646         /*
 647          * Halve as long as the number of empty children in this
 648          * node is above threshold.
 649          */
 650 
 651         max_work = MAX_WORK;
 652         while (tn->bits > 1 &&  max_work-- &&
 653                100 * (tnode_child_length(tn) - tn->empty_children) <
 654                halve_threshold_use * tnode_child_length(tn)) {
 654                halve_threshold_use * tnode_child_length(tn)) {
 655 
 656                 old_tn = tn;
 657                 tn = halve(t, tn);
 658                 if (IS_ERR(tn)) {
 659                         tn = old_tn;
 660 #ifdef CONFIG_IP_FIB_TRIE_STATS
 661                         t->stats.resize_node_skipped++;
 662 #endif
 663                         break;
 664                 }
 665         }
 666 
 667 
 668         /* Only one child remains */
 669         if (tn->empty_children == tnode_child_length(tn) - 1) {
 670 one_child:
 671                 for (i = 0; i < tnode_child_length(tn); i++) {
 672                         struct rt_trie_node *n;
 673 
 674                         n = rtnl_dereference(tn->child[i]);
 675                         if (!n)
 676                                 continue;
 677 
 678                         /* compress one level */
 679 
 680                         node_set_parent(n, NULL);
 681                         tnode_free_safe(tn);
 682                         return n;
 683                 }
 684         }
 685         return (struct rt_trie_node *) tn;
 686 }
 687 
```

### `tnode_put_child_reorg`
英文注释说的很清楚了，主要有两个工作：

1. 更新父节点`tn` 的`full_children` 和 `empty_children`的统计值。
2. 设置`tn`节点的第`i`个子孩子为`n`。

```c
 /*
  * Add a child at position i overwriting the old value.
  * Update the value of full_children and empty_children.
  */

static void tnode_put_child_reorg(struct tnode *tn, int i, struct rt_trie_node *n,
                                  int wasfull)
{
        struct rt_trie_node *chi = rtnl_dereference(tn->child[i]);
        int isfull;

        BUG_ON(i >= 1<<tn->bits);

        /* update emptyChildren */
        if (n == NULL && chi != NULL)
                tn->empty_children++;
        else if (n != NULL && chi == NULL)
                tn->empty_children--;

        /* update fullChildren */
        if (wasfull == -1)
                wasfull = tnode_full(tn, chi);

        isfull = tnode_full(tn, n);
        if (wasfull && !isfull)
                tn->full_children--;
        else if (!wasfull && isfull)
                tn->full_children++;

        if (n)
                node_set_parent(n, tn);

        rcu_assign_pointer(tn->child[i], n);
}
```

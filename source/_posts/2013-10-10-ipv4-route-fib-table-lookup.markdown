---
layout: post
title: "ipv4 route fib table lookup"
date: 2013-10-10 07:34
comments: true
categories: [route]
tags: [ipv4, fib, route]
---

### calltrace
```c
 > ip_rcv_finish
 > > ip_route_input_noref
 > > > ip_route_input_slow
 > > > > fib_lookup
 > > > > > fib_table_lookup with RT_TABLE_LOCAL
 > > > > > fib_table_lookup with RT_TABLE_MAIN
 > > > > ip_mkroute_input
```
<!-- more -->
####
```c
1397 int fib_table_lookup(struct fib_table *tb, const struct flowi4 *flp,
1398                      struct fib_result *res, int fib_flags)
1399 {
1400         struct trie *t = (struct trie *) tb->tb_data;
1401         int ret;
1402         struct rt_trie_node *n;
1403         struct tnode *pn;
1404         unsigned int pos, bits;
1405         t_key key = ntohl(flp->daddr);
1406         unsigned int chopped_off;
1407         t_key cindex = 0;
1408         unsigned int current_prefix_length = KEYLENGTH;
1409         struct tnode *cn;
1410         t_key pref_mismatch;
1411 
1412         rcu_read_lock();
1413 
1414         n = rcu_dereference(t->trie);
1415         if (!n)
1416                 goto failed;
1417 
1418 #ifdef CONFIG_IP_FIB_TRIE_STATS
1419         t->stats.gets++;
1420 #endif
1421 
1422         /* Just a leaf? */
1423         if (IS_LEAF(n)) {
1424                 ret = check_leaf(tb, t, (struct leaf *)n, key, flp, res, fib_flags);
1425                 goto found;
1426         }
1427 
1428         pn = (struct tnode *) n;
1429         chopped_off = 0;
1430 
1431         while (pn) {
1432                 pos = pn->pos;
1433                 bits = pn->bits;
1434 
1435                 if (!chopped_off)
1436                         cindex = tkey_extract_bits(mask_pfx(key, current_prefix_length),
1437                                                    pos, bits);
1438 
1439                 n = tnode_get_child_rcu(pn, cindex);
1440 
1441                 if (n == NULL) {
1442 #ifdef CONFIG_IP_FIB_TRIE_STATS
1443                         t->stats.null_node_hit++;
1444 #endif
1445                         goto backtrace;
1446                 }
1447 
1448                 if (IS_LEAF(n)) {
1449                         ret = check_leaf(tb, t, (struct leaf *)n, key, flp, res, fib_flags);
1450                         if (ret > 0)
1451                                 goto backtrace;
1452                         goto found;
1453                 }
1454 
1455                 cn = (struct tnode *)n;
1456 
1457                 /*
1458                  * It's a tnode, and we can do some extra checks here if we
1459                  * like, to avoid descending into a dead-end branch.
1460                  * This tnode is in the parent's child array at index
1461                  * key[p_pos..p_pos+p_bits] but potentially with some bits
1462                  * chopped off, so in reality the index may be just a
1463                  * subprefix, padded with zero at the end.
1464                  * We can also take a look at any skipped bits in this
1465                  * tnode - everything up to p_pos is supposed to be ok,
1466                  * and the non-chopped bits of the index (se previous
1467                  * paragraph) are also guaranteed ok, but the rest is
1468                  * considered unknown.
1469                  *
1470                  * The skipped bits are key[pos+bits..cn->pos].
1471                  */
1472 
1473                 /* If current_prefix_length < pos+bits, we are already doing
1474                  * actual prefix  matching, which means everything from
1475                  * pos+(bits-chopped_off) onward must be zero along some
1476                  * branch of this subtree - otherwise there is *no* valid
1477                  * prefix present. Here we can only check the skipped
1478                  * bits. Remember, since we have already indexed into the
1479                  * parent's child array, we know that the bits we chopped of
1480                  * *are* zero.
1481                  */
1482 
1483                 /* NOTA BENE: Checking only skipped bits
1484                    for the new node here */
1485 
1486                 if (current_prefix_length < pos+bits) {
1487                         if (tkey_extract_bits(cn->key, current_prefix_length,
1488                                                 cn->pos - current_prefix_length)
1489                             || !(cn->child[0]))
1490                                 goto backtrace;
1491                 }
1492 
1493                 /*
1494                  * If chopped_off=0, the index is fully validated and we
1495                  * only need to look at the skipped bits for this, the new,
1496                  * tnode. What we actually want to do is to find out if
1497                  * these skipped bits match our key perfectly, or if we will
1498                  * have to count on finding a matching prefix further down,
1499                  * because if we do, we would like to have some way of
1500                  * verifying the existence of such a prefix at this point.
1501                  */
1502 
1503                 /* The only thing we can do at this point is to verify that
1504                  * any such matching prefix can indeed be a prefix to our
1505                  * key, and if the bits in the node we are inspecting that
1506                  * do not match our key are not ZERO, this cannot be true.
1507                  * Thus, find out where there is a mismatch (before cn->pos)
1508                  * and verify that all the mismatching bits are zero in the
1509                  * new tnode's key.
1510                  */
1511 
1512                 /*
1513                  * Note: We aren't very concerned about the piece of
1514                  * the key that precede pn->pos+pn->bits, since these
1515                  * have already been checked. The bits after cn->pos
1516                  * aren't checked since these are by definition
1517                  * "unknown" at this point. Thus, what we want to see
1518                  * is if we are about to enter the "prefix matching"
1519                  * state, and in that case verify that the skipped
1520                  * bits that will prevail throughout this subtree are
1521                  * zero, as they have to be if we are to find a
1522                  * matching prefix.
1523                  */
1524 
1525                 pref_mismatch = mask_pfx(cn->key ^ key, cn->pos);
1526 
1527                 /*
1528                  * In short: If skipped bits in this node do not match
1529                  * the search key, enter the "prefix matching"
1530                  * state.directly.
1531                  */
1532                 if (pref_mismatch) {
1533                         /* fls(x) = __fls(x) + 1 */
1534                         int mp = KEYLENGTH - __fls(pref_mismatch) - 1;
1535 
1536                         if (tkey_extract_bits(cn->key, mp, cn->pos - mp) != 0)
1537                                 goto backtrace;
1538 
1539                         if (current_prefix_length >= cn->pos)
1540                                 current_prefix_length = mp;
1541                 }
1542 
1543                 pn = (struct tnode *)n; /* Descend */
1544                 chopped_off = 0;
1545                 continue;
1546 
1547 backtrace:
1548                 chopped_off++;
1549 
1550                 /* As zero don't change the child key (cindex) */
1551                 while ((chopped_off <= pn->bits)
1552                        && !(cindex & (1<<(chopped_off-1))))
1553                         chopped_off++;
1554 
1555                 /* Decrease current_... with bits chopped off */
1556                 if (current_prefix_length > pn->pos + pn->bits - chopped_off)
1557                         current_prefix_length = pn->pos + pn->bits
1558                                 - chopped_off;
1559 
1560                 /*
1561                  * Either we do the actual chop off according or if we have
1562                  * chopped off all bits in this tnode walk up to our parent.
1563                  */
1564 
1565                 if (chopped_off <= pn->bits) {
1566                         cindex &= ~(1 << (chopped_off-1));
1567                 } else {
1568                         struct tnode *parent = node_parent_rcu((struct rt_trie_node *) pn);
1569                         if (!parent)
1570                                 goto failed;
1571 
1572                         /* Get Child's index */
1573                         cindex = tkey_extract_bits(pn->key, parent->pos, parent->bits);
1574                         pn = parent;
1575                         chopped_off = 0;
1576 
1577 #ifdef CONFIG_IP_FIB_TRIE_STATS
1578                         t->stats.backtrack++;
1579 #endif
1580                         goto backtrace;
1581                 }
1582         }
1583 failed:
1584         ret = 1;
1585 found:
1586         rcu_read_unlock();
1587         return ret;
1588 }
1589 EXPORT_SYMBOL_GPL(fib_table_lookup);
```

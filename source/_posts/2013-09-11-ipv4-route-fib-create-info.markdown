---
layout: post
title: "IPv4-route-fib-create-info"
date: 2013-09-11 18:00
comments: true
categories: [route]
ctags: [kernel, route, ipv4, fib]
---

### summary

根据用户参数，创建并返回`fib_create_info`节点。

1. 如果用户参数有问题，则返回空。
   参数检查过程中，会创建一个新的`struct fib_info` 节点，如果参数检查失败，
   该节点会被`free_fib_info`释放（通过rcu模式）
2. 如果存在一个相同配置参数的节点，则返回已有的`struct fib_info` 节点。
   其实已经创建了一个新的`struct fib_info` 节点，该节点会被`free_fib_info`释放（通过rcu模式）同1。
3. 否则，创建一个`struct fib_info` 节点，并初始化，
   节点里相关的`ref`会被increase,同时将该节点链接到 `struct fib_info` 
   相关的3个hash链(`fib_info_hash`, `fib_info_laddrhash`, `fib_info_devhash`)上.
4. 最后返回新建的节点。

<!-- more -->

#### NOTE:
 After this function `struct fib_info *fib_create_info(struct fib_config *cfg)`,
 `struct fib_info`  is only inserted into `fib_info` hash lists, not the fib table(tree).

`struct fib_info *fib_create_info(struct fib_config *cfg)`只是把一个创建了一个
`struct fib_info`节点，并没有真正链接到路由表（fib_tree）里。

```c
 774 struct fib_info *fib_create_info(struct fib_config *cfg) 
 775 { 
 776         int err;
 777         struct fib_info *fi = NULL;
 778         struct fib_info *ofi;
 779         int nhs = 1;
 780         struct net *net = cfg->fc_nlinfo.nl_net;
 781   
 782         if (cfg->fc_type > RTN_MAX)
 783                 goto err_inval;
 784                    
 785         /* Fast check to catch the most weird cases */
 786         if (fib_props[cfg->fc_type].scope > cfg->fc_scope)
 787                 goto err_inval;
 788   
 789 #ifdef CONFIG_IP_ROUTE_MULTIPATH
 790         if (cfg->fc_mp) {
 791                 nhs = fib_count_nexthops(cfg->fc_mp, cfg->fc_mp_len);
 792                 if (nhs == 0)
 793                         goto err_inval;
 794         }
 795 #endif
 796   
 797         err = -ENOBUFS;
 798         if (fib_info_cnt >= fib_info_hash_size) {
 799                 unsigned int new_size = fib_info_hash_size << 1;
 800                 struct hlist_head *new_info_hash;        
 801                 struct hlist_head *new_laddrhash;
 802                 unsigned int bytes;
 803   
 804                 if (!new_size)
 805                         new_size = 16;
 806                 bytes = new_size * sizeof(struct hlist_head *);
 807                 new_info_hash = fib_info_hash_alloc(bytes);
 808                 new_laddrhash = fib_info_hash_alloc(bytes);
 809                 if (!new_info_hash || !new_laddrhash) {  
 810                         fib_info_hash_free(new_info_hash, bytes);
 811                         fib_info_hash_free(new_laddrhash, bytes);
 812                 } else
 813                         fib_info_hash_move(new_info_hash, new_laddrhash, new_size);
 814   
 815                 if (!fib_info_hash_size)
 816                         goto failure;
 817         }    
 818 
 819         fi = kzalloc(sizeof(*fi)+nhs*sizeof(struct fib_nh), GFP_KERNEL);
 820         if (fi == NULL)
 821                 goto failure;
 822         if (cfg->fc_mx) {
 823                 fi->fib_metrics = kzalloc(sizeof(u32) * RTAX_MAX, GFP_KERNEL);
 824                 if (!fi->fib_metrics)
 825                         goto failure;
 826         } else
 827                 fi->fib_metrics = (u32 *) dst_default_metrics;
 828         fib_info_cnt++;
 829 
 830         fi->fib_net = hold_net(net);
 831         fi->fib_protocol = cfg->fc_protocol;
 832         fi->fib_scope = cfg->fc_scope;
 833         fi->fib_flags = cfg->fc_flags;
 834         fi->fib_priority = cfg->fc_priority;
 835         fi->fib_prefsrc = cfg->fc_prefsrc;
 836         fi->fib_type = cfg->fc_type;
 837 
 838         fi->fib_nhs = nhs;
 839         change_nexthops(fi) {
 840                 nexthop_nh->nh_parent = fi;
 841                 nexthop_nh->nh_pcpu_rth_output = alloc_percpu(struct rtable __rcu *);
 842                 if (!nexthop_nh->nh_pcpu_rth_output)
 843                         goto failure;
 844         } endfor_nexthops(fi)
 845         /*  初始化 fib_metric 根据struct nlattr */
 846         if (cfg->fc_mx) {
 847                 struct nlattr *nla;
 848                 int remaining;
 849 
 850                 nla_for_each_attr(nla, cfg->fc_mx, cfg->fc_mx_len, remaining) {
 851                         int type = nla_type(nla);
 852 
 853                         if (type) {
 854                                 u32 val;
 855 
 856                                 if (type > RTAX_MAX)
 857                                         goto err_inval;
 858                                 val = nla_get_u32(nla);
 859                                 if (type == RTAX_ADVMSS && val > 65535 - 40)
 860                                         val = 65535 - 40;
 861                                 if (type == RTAX_MTU && val > 65535 - 15)
 862                                         val = 65535 - 15;
 863                                 fi->fib_metrics[type - 1] = val;
 864                         }
 865                 }
 866         }
 867 
 868         if (cfg->fc_mp) { /* 多个下一跳 */
 869 #ifdef CONFIG_IP_ROUTE_MULTIPATH
 870                 err = fib_get_nhs(fi, cfg->fc_mp, cfg->fc_mp_len, cfg);
 871                 if (err != 0)
 872                         goto failure;
 873                 if (cfg->fc_oif && fi->fib_nh->nh_oif != cfg->fc_oif)
 874                         goto err_inval;
 875                 if (cfg->fc_gw && fi->fib_nh->nh_gw != cfg->fc_gw)
 876                         goto err_inval;
 877 #ifdef CONFIG_IP_ROUTE_CLASSID
 878                 if (cfg->fc_flow && fi->fib_nh->nh_tclassid != cfg->fc_flow)
 879                         goto err_inval;
 880 #endif
 881 #else
 882                 goto err_inval;
 883 #endif
 884         } else { /* 单下一跳 */
 885                 struct fib_nh *nh = fi->fib_nh;
 886 
 887                 nh->nh_oif = cfg->fc_oif;
 888                 nh->nh_gw = cfg->fc_gw;
 889                 nh->nh_flags = cfg->fc_flags;
 890 #ifdef CONFIG_IP_ROUTE_CLASSID
 891                 nh->nh_tclassid = cfg->fc_flow;
 892                 if (nh->nh_tclassid)
 893                         fi->fib_net->ipv4.fib_num_tclassid_users++;
 894 #endif
 895 #ifdef CONFIG_IP_ROUTE_MULTIPATH
 896                 nh->nh_weight = 1;
 897 #endif
 898         }
 899 
 900         if (fib_props[cfg->fc_type].error) { /* 有些特殊的路由，如blackhole, 不允许配置 gw, oif(指定出口 out interface）等参数。*/
 901                 if (cfg->fc_gw || cfg->fc_oif || cfg->fc_mp)
 902                         goto err_inval;
 903                 goto link_it;
 904         } else {
 905                 switch (cfg->fc_type) {
 906                 case RTN_UNICAST:
 907                 case RTN_LOCAL:
 908                 case RTN_BROADCAST:
 909                 case RTN_ANYCAST:
 910                 case RTN_MULTICAST:
 911                         break;
 912                 default:
 913                         goto err_inval;
 914                 }
 915         }
 916 
 917         if (cfg->fc_scope > RT_SCOPE_HOST)
 918                 goto err_inval;
 919 
 920         if (cfg->fc_scope == RT_SCOPE_HOST) { /*  Local address 没有下一跳 */
 921                 struct fib_nh *nh = fi->fib_nh;
 922 
 923                 /* Local address is added. */
 924                 if (nhs != 1 || nh->nh_gw)
 925                         goto err_inval;
 926                 nh->nh_scope = RT_SCOPE_NOWHERE;
 927                 nh->nh_dev = dev_get_by_index(net, fi->fib_nh->nh_oif);
 928                 err = -ENODEV;
 929                 if (nh->nh_dev == NULL)
 930                         goto failure;
 931         } else { /*  非Local address 必须配置下一跳 (gw,oif) */
 932                 change_nexthops(fi) {
 933                         err = fib_check_nh(cfg, fi, nexthop_nh);
 934                         if (err != 0)
 935                                 goto failure;
 936                 } endfor_nexthops(fi)
 937         }
 938 
 939         if (fi->fib_prefsrc) {
 940                 if (cfg->fc_type != RTN_LOCAL || !cfg->fc_dst ||
 941                     fi->fib_prefsrc != cfg->fc_dst)
 942                         if (inet_addr_type(net, fi->fib_prefsrc) != RTN_LOCAL)
 943                                 goto err_inval;
 944         }
 945 
 946         change_nexthops(fi) {
 947                 fib_info_update_nh_saddr(net, nexthop_nh);
 948         } endfor_nexthops(fi)
 949 
 950 link_it:
 951         ofi = fib_find_info(fi); /* 如果已经有相同的 fib，则是否新建的fib(fi), 并返回已有的fib节点  */
 952         if (ofi) {
 953                 fi->fib_dead = 1;
 954                 free_fib_info(fi);
 955                 ofi->fib_treeref++;
 956                 return ofi;
 957         }
 958        /* 至此，这是一个有效的fib， 加锁，增加ref， 并将其链接到 fib 3个hash链里(fib_info_hash, fib_info_laddrhash,fib_info_devhash) */
 959         fi->fib_treeref++;
 960         atomic_inc(&fi->fib_clntref); 
 961         spin_lock_bh(&fib_info_lock);
 962         hlist_add_head(&fi->fib_hash,  
 963                        &fib_info_hash[fib_info_hashfn(fi)]);
 964         if (fi->fib_prefsrc) {
 965                 struct hlist_head *head;
 966 
 967                 head = &fib_info_laddrhash[fib_laddr_hashfn(fi->fib_prefsrc)];
 968                 hlist_add_head(&fi->fib_lhash, head);
 969         }
 970         change_nexthops(fi) {
 971                 struct hlist_head *head;
 972                 unsigned int hash;
 973 
 974                 if (!nexthop_nh->nh_dev)
 975                         continue;
 976                 hash = fib_devindex_hashfn(nexthop_nh->nh_dev->ifindex);
 977                 head = &fib_info_devhash[hash];
 978                 hlist_add_head(&nexthop_nh->nh_hash, head);
 979         } endfor_nexthops(fi)
 980         spin_unlock_bh(&fib_info_lock);
 981         return fi;
 982 
 983 err_inval:
 984         err = -EINVAL;
 985 
 986 failure:
 987         if (fi) {
 988                 fi->fib_dead = 1;
 989                 free_fib_info(fi);
 990         }
 991 
 992         return ERR_PTR(err);
 993 }
```


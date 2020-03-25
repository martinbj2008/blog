---
layout: post
title: "IPv4 route fib table insert"
date: 2013-09-11 17:19
comments: true
categories: [route]
tags: [kernel, IPv4, fib]
---


#### summary
todo!! 

<!-- more -->
 
```c
1168 /*
1169  * Caller must hold RTNL.
1170  */
1171 int fib_table_insert(struct fib_table *tb, struct fib_config *cfg)
1172 {
1173         struct trie *t = (struct trie *) tb->tb_data;
1174         struct fib_alias *fa, *new_fa;
1175         struct list_head *fa_head = NULL;
1176         struct fib_info *fi;
1177         int plen = cfg->fc_dst_len;
1178         u8 tos = cfg->fc_tos;
1179         u32 key, mask;
1180         int err;
1181         struct leaf *l;
1182 
1183         if (plen > 32)
1184                 return -EINVAL;
1185 
1186         key = ntohl(cfg->fc_dst);
1187 
1188         pr_debug("Insert table=%u %08x/%d\n", tb->tb_id, key, plen);
1189 
1190         mask = ntohl(inet_make_mask(plen));
1191 
1192         if (key & ~mask)
1193                 return -EINVAL;
1194 
1195         key = key & mask;
1196 
1197         fi = fib_create_info(cfg);
1198         if (IS_ERR(fi)) {
1199                 err = PTR_ERR(fi);
1200                 goto err;
1201         }
1202 
1203         l = fib_find_node(t, key);
1204         fa = NULL;
1205 
1206         if (l) {
1207                 fa_head = get_fa_head(l, plen);
1208                 fa = fib_find_alias(fa_head, tos, fi->fib_priority);
1209         }
1210 
1211         /* Now fa, if non-NULL, points to the first fib alias
1212          * with the same keys [prefix,tos,priority], if such key already
1213          * exists or to the node before which we will insert new one.
1214          *
1215          * If fa is NULL, we will need to allocate a new one and
1216          * insert to the head of f.
1217          *
1218          * If f is NULL, no fib node matched the destination key
1219          * and we need to allocate a new one of those as well.
1220          */
1221 
1222         if (fa && fa->fa_tos == tos &&
1223             fa->fa_info->fib_priority == fi->fib_priority) {
1224                 struct fib_alias *fa_first, *fa_match;
1225 
1226                 err = -EEXIST;
1227                 if (cfg->fc_nlflags & NLM_F_EXCL)
1228                         goto out;
1229 
1230                 /* We have 2 goals:
1231                  * 1. Find exact match for type, scope, fib_info to avoid
1232                  * duplicate routes
1233                  * 2. Find next 'fa' (or head), NLM_F_APPEND inserts before it
1234                  */
1235                 fa_match = NULL;
1236                 fa_first = fa;
1237                 fa = list_entry(fa->fa_list.prev, struct fib_alias, fa_list);
1238                 list_for_each_entry_continue(fa, fa_head, fa_list) {
1239                         if (fa->fa_tos != tos)
1240                                 break;
1241                         if (fa->fa_info->fib_priority != fi->fib_priority)
1242                                 break;
1243                         if (fa->fa_type == cfg->fc_type &&
1244                             fa->fa_info == fi) {
1245                                 fa_match = fa;
1246                                 break;
1247                         }
1248                 }
1249 
1250                 if (cfg->fc_nlflags & NLM_F_REPLACE) {
1251                         struct fib_info *fi_drop;
1252                         u8 state;
1253 
1254                         fa = fa_first;
1255                         if (fa_match) {
1256                                 if (fa == fa_match)
1257                                         err = 0;
1258                                 goto out;
1259                         }
1260                         err = -ENOBUFS;
1261                         new_fa = kmem_cache_alloc(fn_alias_kmem, GFP_KERNEL);
1262                         if (new_fa == NULL)
1263                                 goto out;
1264 
1265                         fi_drop = fa->fa_info;
1266                         new_fa->fa_tos = fa->fa_tos;
1267                         new_fa->fa_info = fi;
1268                         new_fa->fa_type = cfg->fc_type;
1269                         state = fa->fa_state;
1270                         new_fa->fa_state = state & ~FA_S_ACCESSED;
1271 
1272                         list_replace_rcu(&fa->fa_list, &new_fa->fa_list);
1273                         alias_free_mem_rcu(fa);
1274 
1275                         fib_release_info(fi_drop);
1276                         if (state & FA_S_ACCESSED)
1277                                 rt_cache_flush(cfg->fc_nlinfo.nl_net);
1278                         rtmsg_fib(RTM_NEWROUTE, htonl(key), new_fa, plen,
1279                                 tb->tb_id, &cfg->fc_nlinfo, NLM_F_REPLACE);
1280 
1281                         goto succeeded;
1282                 }
1283                 /* Error if we find a perfect match which
1284                  * uses the same scope, type, and nexthop
1285                  * information.
1286                  */
1287                 if (fa_match)
1288                         goto out;
1289 
1290                 if (!(cfg->fc_nlflags & NLM_F_APPEND))
1291                         fa = fa_first;
1292         }
1293         err = -ENOENT;
1294         if (!(cfg->fc_nlflags & NLM_F_CREATE))
1295                 goto out;
1296 
1297         err = -ENOBUFS;
1298         new_fa = kmem_cache_alloc(fn_alias_kmem, GFP_KERNEL);
1299         if (new_fa == NULL)
1300                 goto out;
1301 
1302         new_fa->fa_info = fi;
1303         new_fa->fa_tos = tos;
1304         new_fa->fa_type = cfg->fc_type;
1305         new_fa->fa_state = 0;
1306         /*
1307          * Insert new entry to the list.
1308          */
1309 
1310         if (!fa_head) {
1311                 fa_head = fib_insert_node(t, key, plen);
1312                 if (unlikely(!fa_head)) {
1313                         err = -ENOMEM;
1314                         goto out_free_new_fa;
1315                 }
1316         }
1317 
1318         if (!plen)
1319                 tb->tb_num_default++;
1320 
1321         list_add_tail_rcu(&new_fa->fa_list,
1322                           (fa ? &fa->fa_list : fa_head));
1323 
1324         rt_cache_flush(cfg->fc_nlinfo.nl_net);
1325         rtmsg_fib(RTM_NEWROUTE, htonl(key), new_fa, plen, tb->tb_id,
1326                   &cfg->fc_nlinfo, 0);
1327 succeeded:
1328         return 0;
1329 
1330 out_free_new_fa:
1331         kmem_cache_free(fn_alias_kmem, new_fa);
1332 out:
1333         fib_release_info(fi);
1334 err:
1335         return err;
1336 }
```


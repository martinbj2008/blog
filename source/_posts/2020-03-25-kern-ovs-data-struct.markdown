---
layout: post
title: "内核OVS的学习总结"
date: 2020-03-25 18:56:08 +0800
comments: true
categories:
---


-------------------
[TOC]

OVS里最重要的几个元素：vport，flow，datapath。
其中datapth是vport和flow的桥梁。

![kernel ovs 核心结构体及其关联](/images/ovs_kernl_full.png)

## VPORT
内核包含多个datapath（brige），上面包含一个或者多个vport。
其中一个VPORT表示一个端口，一个vport只能归于一个特定的datapath。
每个vport有自己的`type`,  对应不同的`vport ops`.
每个内核网口被注册为vport的时候。

<!--more-->
###  VPORT结构体
```
 58 /**
 59  * struct vport - one port within a datapath
 60  * @dev: Pointer to net_device.
 61  * @dp: Datapath to which this port belongs.
 62  * @upcall_portids: RCU protected 'struct vport_portids'.
 63  * @port_no: Index into @dp's @ports array.
 64  * @hash_node: Element in @dev_table hash table in vport.c.
 65  * @dp_hash_node: Element in @datapath->ports hash table in datapath.c.
 66  * @ops: Class structure.
 67  * @detach_list: list used for detaching vport in net-exit call.
 68  * @rcu: RCU callback head for deferred destruction.
 69  */
 70 struct vport {
 71         struct net_device *dev;
 72         struct datapath *dp;
 73         struct vport_portids __rcu *upcall_portids;
 74         u16 port_no;
 75
 76         struct hlist_node hash_node;
 77         struct hlist_node dp_hash_node;
 78         const struct vport_ops *ops;
 79
 80         struct list_head detach_list;
 81         struct rcu_head rcu;
 82 };
```

### vport & netdevice：
+ dev: 当一个网口`struct net_device`被加入到datapath的时候，会创建一个对应的vport，并通过vport->dev指向这个网口。
+ 对应的网口的`rx_handler`被设置为`netdev_frame_hook`, `rx_handler_data`设置为vport本身。
```
102         err = netdev_rx_handler_register(vport->dev, netdev_frame_hook,
103                                          vport);
```
### vport & datapath
+ 一个datapath相当于一个bridge（OVS用户空间实现里，把多个用户空间的网桥整合到一个datapath里了）。
+ 一个datapath下可以有多个vport，但一个vport只能归属于一个datapath。
+ 同一个datapath下的所有vport通过vport下的`dp_hash_node`, 根据`vport->port_no`组成一个hash桶.  这个hash桶的链表头放在了datapath下的`struct hlist_head *ports`。详细过程见函数`static struct vport *new_vport(const struct vport_parms *parms)`
+ 内核ovs下所有的vport还会有一个全局的hash链表。根据vport的名字做关键字计算hash，放到一个全局的hash链表`static struct hlist_head *dev_table`。注：(vport跟其对应的netdevice名字一致), 
+ 
### vport & `struct vport_ops`
每个vport在创建时候都需要指定其对应的port 类型，每个port类型都有其独有的port ops。通过`vport->ops`指向其对应的` struct vport_ops`
内核里有多种类型的vport，不同类型vport对应的port ops组成一个单链表`vport_ops_list`。


## flow
ovs 里的flow通过match-action的形式定义了报文的处理路径。
flow的match部分是通过一个key+mask的形式进行匹配的。
key有很多元素，比如vni，src ip，mac，tunnel等信息。
mask跟key差不多的，只是多了一个range，用于指明mask里的key具体那个范围的值有效。即key从start到end部分是有效信息。
ovs收到报文后会解析每个报文的key，然后根据报文的key和每个flow的相与，得到的掩过的key跟flow里的key进行比较，确定是不是满足flow的match条件。
```
161 struct sw_flow_key_range {
162         unsigned short int start;
163         unsigned short int end;
164 };
```

```
166 struct sw_flow_mask {
167         int ref_count;
168         struct rcu_head rcu;
169         struct list_head list;
170         struct sw_flow_key_range range;
171         struct sw_flow_key key;
172 };
```

### flow的管理
+ 每个datapath下有一个flow table。
+ 一个flow table相当于一个flow group，下面有多条flow
每个flow table下有个hash链，每个flow根据其key的hash，存放到对应的hash桶下的链表里。

内核版ovs跟dpdk ovs的一个区别是，table下的所有的flow放到一个hash链表里，没有根据mask再做区分。所有flow的mask类型被放到一个单独的链表里。
当报文来查找规则的时候，会根据mask list里的mask逐个做`mask`操作，
根据掩后的key，做hash去hash链里查找对应的flow。
重复`mask list`直到mask耗尽，或者找到匹配的flow。

```
205 struct sw_flow {
206         struct rcu_head rcu;
207         struct {
208                 struct hlist_node node[2];
209                 u32 hash;
210         } flow_table, ufid_table;
211         int stats_last_writer;          /* CPU id of the last writer on
212                                          * 'stats[0]'.
213                                          */
214         struct sw_flow_key key;
215         struct sw_flow_id id;
216         struct cpumask cpu_used_mask;
217         struct sw_flow_mask *mask;
218         struct sw_flow_actions __rcu *sf_acts;
219         struct sw_flow_stats __rcu *stats[]; /* One for each CPU.  First one
220                                            * is allocated at flow creation time,
221                                            * the rest are allocated on demand
222                                            * while holding the 'stats[0].lock'.
223                                            */
224 };
```

## datapath
datapath从管理上看类似传统的网桥，是ovs里用来链接vport和flow的一个汇聚点。
所有的vport和flow都会归属到某一个特定的网桥上。
如之前所述，

+ `struct list_head list_node`：内核里所有的DP通过一个双向链接串接起来。
+ `struct hlist_head *ports`：通过一个hash链表管理当前dp下所有的vport。
+ `struct flow_table table`: 管理当前dp下的所有flow流。


```
 49 /**
 50  * struct datapath - datapath for flow-based packet switching
 51  * @rcu: RCU callback head for deferred destruction.
 52  * @list_node: Element in global 'dps' list.
 53  * @table: flow table.
 54  * @ports: Hash table for ports.  %OVSP_LOCAL port always exists.  Protected by
 55  * ovs_mutex and RCU.
 56  * @stats_percpu: Per-CPU datapath statistics.
 57  * @net: Reference to net namespace.
 58  * @max_headroom: the maximum headroom of all vports in this datapath; it will
 59  * be used by all the internal vports in this dp.
 60  *
 61  * Context: See the comment on locking at the top of datapath.c for additional
 62  * locking information.
 63  */
 64 struct datapath {
 65         struct rcu_head rcu;
 66         struct list_head list_node;
 67
 68         /* Flow table. */
 69         struct flow_table table;
 70
 71         /* Switch ports. */
 72         struct hlist_head *ports;
 73
 74         /* Stats. */
 75         struct dp_stats_percpu __percpu *stats_percpu;
 76
 77         /* Network namespace ref. */
 78         possible_net_t net;
 79
 80         u32 user_features;
 81
 82         u32 max_headroom;
 83
 84         /* Switch meters. */
 85         struct hlist_head *meters;
 86 };
```

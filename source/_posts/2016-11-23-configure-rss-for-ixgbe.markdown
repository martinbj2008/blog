---
layout: post
title: "configure rss for ixgbe"
date: 2016-11-23 15:33:07 +0800
comments: true
categories: 
---

### 配置ixgbe网卡的RSS

```
ethtool -K em1 ntuple on
ethtool --show-ntuple em1
ethtool --config-ntuple em1  flow-type tcp4 dst-port 60001 action 1
ethtool --config-ntuple em1  flow-type tcp4 dst-port 60002 action 2
ethtool --show-ntuple em1
```
<!-- more -->

清除命令
```
ethtool --config-ntuple em1  delete 2042
ethtool --config-ntuple em1  delete 2043
ethtool -K em1 ntuple off
```

查看效果
```
[root@localhost ~]# ethtool  -S em1|grep fdir
     fdir_match: 156292024
     fdir_miss: 357540
     fdir_overflow: 43
```

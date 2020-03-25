---
layout: post
title: "netlink grab"
date: 2013-03-11 00:00
comments: true
categories: [netlink]
tags: [kernel, netlink]
---

对`nl_table`的操作：    
读操作: `netlink_lock_table` 和 `netlink_unlock_table`   
写操作`netlink_table_grab` 和 `netlink_table_ungrab`     

原子变量`nl_table_users` 用来保存对`nl_table`的读者引用计数。

只要没有进行的写者，即使有n个写者在等待，m个读者也可以同时读。
当没有任何读者时，写着才可以获得权限。
一旦一个写着获得权限，所有的读者和写者都得等待。

<!-- more -->

### `nl_table_lock`
保证了      
1. 读写之间的互斥   
2. 写者之间的互斥   

### `nl_table_users`  
读者的引用计数。


感觉很类是读写锁，只是写着可以睡眠。
#data struct
134 static DECLARE_WAIT_QUEUE_HEAD(nl_table_wait);
138 static DEFINE_RWLOCK(nl_table_lock);
139 static atomic_t nl_table_users = ATOMIC_INIT(0);

#### 读操作上锁：
a. 获取锁`nl_table_lock`的读权限，
b. 访问数增加1.
c. 释放读锁。

 229 static inline void
 230 netlink_lock_table(void)
 231 {       
 232         /* read_lock() synchronizes us to netlink_table_grab */
 233  
 234         read_lock(&nl_table_lock);
 235         atomic_inc(&nl_table_users);
 236         read_unlock(&nl_table_lock);
 237 }  

#### 读操作开锁：
 nl_table_users减一，
 判断nl_table_users == 0？
 YES, nl_table_users为0，表示没有读者, 调用wakeup唤醒等待的写操作。
 NO, --return.

 239 static inline void
 240 netlink_unlock_table(void)
 241 { 
 242         if (atomic_dec_and_test(&nl_table_users))
 243                 wake_up(&nl_table_wait);
 244 }

#### 写操作上锁:
1.而写操作首先获取锁`nl_table_lock`的写权限，
如果当前用户个数为0，
（幸运啊）那么后续的所有操作（读和写)都被阻塞.因为`nl_table_lock`被锁上了。

如果用户数不为零，那么存在nl_table_users个读者和m（m>=0)个等待的写着。
创建一个等待队列，等待其他的读或者写的完成，同时自己通过schedule放弃cpu。

再次得到调度时，首先获取写锁`nl_table_lock`，重复上一步的判断。
[netlink_table_grab](https://gist.github.com/martinbj2008/b878b0a3d55533e7073f)

#### 写操作开锁：
释放写锁。
唤醒等待队列。
 222 void netlink_table_ungrab(void)
 223         __releases(nl_table_lock)
 224 { 
 225         write_unlock_irq(&nl_table_lock);        
 226         wake_up(&nl_table_wait);
 227 } 


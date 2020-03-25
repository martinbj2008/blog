---
layout: post
title: "how to use git bisect"
date: 2013-08-23 10:37
comments: true
categories: 
---


## usage:
In version management, we often meet this case:

```
 When a line/file is deleted for git repo?
 or 
 who did it ?
```
In this case, we need `git bisect`.

1. `git bisect start`
2. `git bisect good commit_id1`
3. `git bisect bad commit_id2`
4. `git bisect bad/good` 

repeat to step 4, until finish. 

more auto method for step4 is run with `git bisect run command`.

by the way `git blame` is used to check the added line/file.

<!-- more -->

## example

in current kernel(v3.0) the IPv4 route has remove the file `fib_hash.c`,
which stores the old IPv4 route method.
while it still is seen in kernel(v2.6.36).

let find out which kernel version remove it and the related commmit ID.

4 commonds will run:
```
git bisect start
git bisect good v2.6.36  
git bisect bad v3.0
git bisect run ls net/ipv4/fib_hash.c
```

```
➜  linux git:(1f922d0) ✗ git bisect start
Checking out files: 100% (42337/42337), done.
Previous HEAD position was 1f922d0... Merge branch 'for_linus' of git://git.kernel.org/pub/scm/linux/kernel/git/jwessel/linux-2.6-kgdb
Switched to branch 'master'
➜  linux git:(master) ✗ git bisect good v2.6.36  
➜  linux git:(master) ✗ git bisect bad v3.0
Bisecting: 21736 revisions left to test after this (roughly 14 steps)
[078a198906c796981f93ff100c210506e91aade5] x86-64, NUMA: Don't assume phys node 0 is always online in numa_emulation()
➜  linux git:(078a198) ✗ git bisect run ls net/ipv4/fib_hash.c
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 10868 revisions left to test after this (roughly 13 steps)
[c0d289b3e59577532c45ee9110ef81bd7b341272] [SCSI] scsi_dh_alua: Attach to UNAVAILABLE/OFFLINE AAS devices
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
Bisecting: 5450 revisions left to test after this (roughly 12 steps)
[02e4c627d862427653fc088ce299746ea7d85600] Merge branch 'for-linus/2639/i2c-1' of git://git.fluff.org/bjdooks/linux
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
Bisecting: 3000 revisions left to test after this (roughly 11 steps)
[ceda86a108671294052cbf51660097b6534672f5] bonding: enable netpoll without checking link status
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
Bisecting: 1207 revisions left to test after this (roughly 10 steps)
[63d8ea7f93e1fb9d1aa9509ab3e1a71199245c80] net: Forgot to commit net/core/dev.c part of Jiri's ->rx_handler patch.
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
Bisecting: 604 revisions left to test after this (roughly 9 steps)
[29e1846a6ba84e0c6e257dd5b1231ed53b98fe9b] Merge branch 'fec' of git://git.pengutronix.de/git/ukl/linux-2.6
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
Bisecting: 311 revisions left to test after this (roughly 8 steps)
[123b9731b14f49cd41c91ed2b6c31e515615347c] ipv4: Rename fib_hash_* locals in fib_semantics.c
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
Bisecting: 145 revisions left to test after this (roughly 7 steps)
[84c49d8c3e4abefb0a41a77b25aa37ebe8d6b743] veth: remove unneeded ifname code from veth_newlink()
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 71 revisions left to test after this (roughly 6 steps)
[62362dee83c2a50465cc64ba701ebd741996ec80] Merge branch 'wireless-next-2.6' of git://git.kernel.org/pub/scm/linux/kernel/git/iwlwifi/iwlwifi-2.6
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 35 revisions left to test after this (roughly 5 steps)
[b8dad61cc74b9ec71052e2a0e1c5119c65d166da] ipv4: If fib metrics are default, no need to grab ref to FIB info.
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 17 revisions left to test after this (roughly 4 steps)
[1bef68e3f5d25e17adc5232dc0ad7c0ea0188374] bnx2x: Add CMS functionality for 848x3
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 8 revisions left to test after this (roughly 3 steps)
[091b948306d2628320e77977eb7ae4a757b12180] batman-adv: Merge README of v2011.0.0 release
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 4 revisions left to test after this (roughly 2 steps)
[5b4704419cbd0b7597a91c19f9e8e8b17c1af071] ipv4: Remember FIB alias list head and table in lookup results.
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 2 revisions left to test after this (roughly 1 step)
[2ba451421b23636c45fabfa522858c5c124b3673] bnx2x, cnic: Consolidate iSCSI/FCoE shared mem logic in bnx2x
running ls net/ipv4/fib_hash.c
net/ipv4/fib_hash.c
Bisecting: 0 revisions left to test after this (roughly 1 step)
[5348ba85a02ffe80a8af33a524b6610966760d3d] ipv4: Update some fib_hash centric interface names.
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
Bisecting: 0 revisions left to test after this (roughly 0 steps)
[3630b7c050d9c3564f143d595339fc06b888d6f3] ipv4: Remove fib_hash.
running ls net/ipv4/fib_hash.c
ls: cannot access net/ipv4/fib_hash.c: No such file or directory
3630b7c050d9c3564f143d595339fc06b888d6f3 is the first bad commit
commit 3630b7c050d9c3564f143d595339fc06b888d6f3
Author: David S. Miller <davem@davemloft.net>
Date:   Tue Feb 1 15:15:39 2011 -0800

    ipv4: Remove fib_hash.
    
    The time has finally come to remove the hash based routing table
    implementation in ipv4.
    
    FIB Trie is mature, well tested, and I've done an audit of it's code
    to confirm that it implements insert, delete, and lookup with the same
    identical semantics as fib_hash did.
    
    If there are any semantic differences found in fib_trie, we should
    simply fix them.
    
    I've placed the trie statistic config option under advanced router
    configuration.
    
    Signed-off-by: David S. Miller <davem@davemloft.net>
    Acked-by: Stephen Hemminger <shemminger@vyatta.com>

:040000 040000 946c438b3b305abbdf017b60a09a2f3657196c44 90b48c05fad8a0decb0121d0219825997cc07b76 M	net
bisect run success
➜  linux git:(3630b7c) ✗ 
```

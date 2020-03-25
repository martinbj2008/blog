---
layout: post
title: "ftrace study"
date: 2014-10-21 14:45
comments: true
categories: [debug]
tags: [ftrace]
---

### test case
We found ixgbe  rx softirq  poll function `ixgbe_poll` was called even without pkt coming.
how to prove it and who call it?

### analysis
By browse source `ixgbe_poll` is called, it should be done by napi schedule.
If this is true, `__napi_schedule` should be called.

<!-- more -->
### use ftrace to locate:

##### prepare debugfs
```c
mount -t debugfs none /sys/kernel/debug/
cd  /sys/kernel/debug/tracing
```

####step1: who call `__napi_schedule`
```c
echo function > current_tracer
echo 1 > options/func_stack_trace
echo __napi_schedule > set_ftrace_filter
cat set_ftrace_filter
cat trace  > ~/a
```

#### step2: who call `ixgbe_msix_clean_rings`
```
echo function > current_tracer
echo 1 > options/func_stack_trace
echo ixgbe_msix_clean_rings > set_ftrace_filter
cat set_ftrace_filter
cat trace  > ~/a
```

#### ftrace document
Debugging the kernel using Ftrace
part 1
http://lwn.net/Articles/365835/

part 2
http://lwn.net/Articles/366796/

Secrets of the Ftrace function tracer
http://lwn.net/Articles/370423/

Debugging Linux Kernel by Ftrace by AceLan Kao
http://people.canonical.com/~acelan/coscup-2010/Debugging%20Linux%20Kernel%20by%20Ftrace.pdf

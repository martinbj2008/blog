---
layout: post
title: "Qdisc running flag"
date: 2014-02-08 11:26
comments: true
categories: [netcore]
tags:	[net, qdisc]
---

### Summary
In `struct Qdisc`, there are two similar fileds.
running flag is stored in **`__state`** of `struct Qdisc`, NOT **`state`**.
Every time, when we send a packet from qdisc, the `running` flag is 
set by `qdisc_run_begin`, and after that, it is removed by `qdisc_run_end`.

```c
 84         unsigned long           state;
 ...
 87         unsigned int            __state;
```
#### todo
 why need busylock?

<!-- more -->

#### The values of `state`.
```c
 24 enum qdisc_state_t {
 25         __QDISC_STATE_SCHED,
 26         __QDISC_STATE_DEACTIVATED,
 27         __QDISC_STATE_THROTTLED,
 28 };
```
#### The value of `__state`.
```c
 33 enum qdisc___state_t {
 34         __QDISC___STATE_RUNNING = 1,
 35 };
```

## Running flag

### check Qdisc running
```c
 96 static inline bool qdisc_is_running(const struct Qdisc *qdisc)
 97 {
 98         return (qdisc->__state & __QDISC___STATE_RUNNING) ? true : false;
 99 }
```

### Set Qdisc running
```c
101 static inline bool qdisc_run_begin(struct Qdisc *qdisc)
102 {
103         if (qdisc_is_running(qdisc))
104                 return false;
105         qdisc->__state |= __QDISC___STATE_RUNNING;
106         return true;
107 }
```
### Unset Qdisc running 
```c
109 static inline void qdisc_run_end(struct Qdisc *qdisc)
110 {
111         qdisc->__state &= ~__QDISC___STATE_RUNNING;
112 }
```

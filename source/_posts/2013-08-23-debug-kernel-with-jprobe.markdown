---
layout: post
title: "debug kernel with jprobe"
date: 2013-08-23 11:13
comments: true
categories: [debug]
tags: [kernel, debug]
---

## kprobe is a useful tool when debug kernel.

It can directly hook the related function of kernel,
before or after it run.

It also can be used to patch kernel in some special case.

<!-- more -->

## example
We can update kernel stat before kernel show it.

```c
  1 #include <linux/version.h>
  2 #include <linux/module.h>
  3 #include <linux/kallsyms.h>
  4 #include <linux/kprobes.h>
  5 
  6 static int martin_snmp_seq_show(struct seq_file *seq, void *v)
  7 {
  8         printk(KERN_INFO "%s: called\n", __func__);
  9 
 10         jprobe_return();
 11         return 0;
 12 }
 13 
 14 static int martin_snmp6_seq_show(struct seq_file *seq, void *v)
 15 {  
 16         printk(KERN_INFO "%s: called\n", __func__);
 17    
 18         jprobe_return();
 19         return 0;
 20 }
 21 
 22 static struct jprobe martin_snmp_jprobe = {
 23         .entry = martin_snmp_seq_show,
 24         .kp = {
 25                 .symbol_name = "snmp_seq_show",          
 26         },
 27 };
 28 
 29 static struct jprobe martin_snmp6_jprobe = {
 30         .entry = martin_snmp6_seq_show,
 31         .kp = {
 32                 .symbol_name = "snmp6_seq_show",         
 33         },
 34 };
 35 
 36 static struct jprobe* martin_jprobes[] = {
 37         &martin_snmp_jprobe,
 38         &martin_snmp6_jprobe,
 39 };
 40 
 41 int __init martin_jprobe_init(void)
 42 {
 43         const int cnt = sizeof(martin_jprobes)/sizeof(struct jprobe *);
 44 
 45         if (register_jprobes(martin_jprobes, cnt) < 0) {
 46                 printk(KERN_INFO "FPS: register_jprobe failed !!!\n");
 47                 return -1;
 48         }
 49 
 50         printk(KERN_INFO "FPS: ready\n");        
 51         return 0;
 52 }
 53 
 54 void __exit martin_jprobe_exit(void)
 55 {
 56         const int cnt = sizeof(martin_jprobes)/sizeof(struct jprobe *);
 57 
 58         unregister_jprobes(martin_jprobes, cnt);
 59 
 60         printk(KERN_INFO "FPS: coloc martin module exit.\n");
 61 }
 62 
 63 module_init(martin_jprobe_init);
 64 module_exit(martin_jprobe_exit);
 65 
 66 MODULE_DESCRIPTION("MartinJPROBE");
 67 MODULE_LICENSE("GPL");
```

---
layout: post
title: "LFS中Binutils,GCC,Glibc三者之间的关系"
date: 2012-06-01 00:00
comments: true
categories: [gcc]
tags: [glibc, gcc, toolchain]
---

原文地址：LFS中Binutils,GCC,Glibc三者之间的关系
[http://blog.chinaunix.net/uid-20431728-id-2752867.html]
<!-- more -->

1. binutils有一个很重要的目的是为了生成LD，标准连接器。以及as汇编器，还有readelf等等。
2. gcc，生成gcc编译器
3. head头文件，必要的头文件支持，变量和函数的申明.
4. glibc，利用新的头文件以及新的binutils程序，生成glibc，其中有大名顶顶的ld-linux.so动态加载器。其中/etc/ld.so.conf文件的作用是库文件的搜索路径，默认情况下，编译器只会查询/lib和/usr/lib这两个目录下的库文件。
5. 在上述编译过程中，常出现—libexecdir的参数，表示将程序在编译过程中将生成的.so和.a文件放到该目录内
6. 调整工具链，即启用新工具链，新的/bin/ld，以及新的/lib/ld-linux.so.2。其中ld-linux.so.2连接的重新定位依靠修改gcc --print-file specs文件来实现。
7. 利用新的库文件和工具链，重新安装GCC和binutils，以彻底摆脱宿主系统的控制。


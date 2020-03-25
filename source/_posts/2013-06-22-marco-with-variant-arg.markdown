---
layout: post
title: "gcc参数可变的宏定义"
date: 2007-06-12 09:06
comments: true
categories: [gcc]
tags: [marco, arg, variant]
---

`#define LOGSTRINGS(fm, ...) printf(fm,__VA_ARGS__)`

使用方法
`LOGSTRINGS("hello, %d ", 10);`

但是必须有第二个参数！否则编译无法通过。


改进
`#define LOGSTRINGS(fm, ...) printf(fm, ##__VA_ARGS__)`	
`LOGSTRINGS("hello, world\n");`	

最简单的方法	
`#define LOGSTRINGS(...) printf（__VA_ARGS__)`

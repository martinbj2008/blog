---
layout: post
title: "关闭linux的beep声音"
date: 2007-08-01 00:00
comments: true
categories: [others]
tags: [beep, bash]
---

在Linux/Unix下的命令行下，经常会用到shell的自动补齐，但是同时也带来了烦人的beep声，煞是刺耳。  
有2种方法可以将它踢掉：

1. 修改配置文件  
确保/etc/inputrc中有如下一行未被注释  
`set bell-style none`  
修改后重新登录.  
如果使用vim还要修改vimrc,在~/.vimrc中添加一行:  
`set vb t_vb=`

2.编译内核时关闭相关的选项  
使/usr/src/linux/.config文件中包含  
`CONFIG_SPEAKUP_DEFAULT=”n”`

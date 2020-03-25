---
layout: post
title: "How to make selected tab in terminal more prominent"
date: 2013-10-07 18:57
comments: true
categories: [others]
tags: [ubuntu, terminal, gtk]
---
On ubuntu 13.04, create the file ~/.config/gtk-3.0/gtk.css with following lines

```
martin@ubuntu:~$ cat ~/.config/gtk-3.0/gtk.css 
TerminalWindow .notebook tab:active {
    background-color: #0000FF;	
    }
martin@ubuntu:~$ 
```


---
layout: post
title: "gcc attribute"
date: 2007-06-12 00:00
comments: true
categories: [gcc]
tags: [gcc]
---

<!-- more -->

1. -I 指定头文件搜索路径（I 表include） 
如 $gcc -c hello.c -o hello.o -I/usr/include 
2.-L 指定要连接的库所在的目录 
-l 指定要连接的库的名字 
如$gcc main.o -L/usr/lib -lqt -o hello 
3. -D 定义宏(D-define) 
-D定义宏有两种情况，一种是-DMACRO 相当于程序中使用#define MACRO 另外可以-DMACRO=A 相当于程序中使用#define MACRO A 这只是一个编绎参数，在连接时没有意义 
如： $gcc -c hello.c -o hello.o -DDEBUG 
上面为hello.c定义了一个DEBUG宏，某些情况下使用-D 代替直接在文件中使用#define，也是为了避免修改源代码双。例如一个程序希望在开发调试的时候能打印出调试信息，而正式发布的时候就不用打印了，而且发布前不用修改源代码双。可以这样 

#ifdefine DEBUG 
printf("debug message\n"); 
#endif 
对于这段代码，平时调试的时候就加上-DDEBUG 发布时不用-D选项 
与之对应的是-UMACRO参数，相当于#undef MACRO，取消宏定义 

4. -g 生成调试信息 
-g生成调试信息，这对使用gdb进行调试是必须的。带有调试信息的文件要比普通文件要大，但不影响运行，可以用strip命令除于其中的调试信息 

5. -c指于gcc只进行编绎，不连接 

6. -ansi 指示gcc只支持ansi c标准语法 

7. -o 指定输出文件名 

8. -O 指定优化处理 
-O0不优化 -O1或-O 一级优化 -O2 二级优化...-O3,-O4 
级别越高，，代码越优，编绎时间越长。 

9. -m486 针对特定的目标计算机进行优化，默认是386 

10. -w 关闭编译器警告信息

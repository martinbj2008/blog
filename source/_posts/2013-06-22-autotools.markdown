---
layout: post
title: "Autotools"
date: 2007-06-22 00:00
comments: true
categories: [make]
---

##步骤简要表述：
1. 创建 configure.ac 文件
2. 在根目录及各个子目录下依次创建Makefile.am文件。
3. 使用命令“autoreconf --install” 命令自动生成confiugre文件
4. 执行新生成的脚本 configure
5. 执行make文件。

<!-- more -->

###步骤1：创建一个configure.ac
内容如下：
# Process this file with autoconf to produce a configure script.
AC_PREREQ(2.59)
#需要手动修改
AC_INIT(realview, 1.0, )
#需要手动添加，选项foreign放宽对软件的发行要求。
AM_INIT_AUTOMAKE(-Wall -Werror foreign)
AC_CONFIG_SRCDIR([config.h.in])
AC_CONFIG_HEADER([config.h])
# Checks for programs.
AC_PROG_CXX
AC_PROG_CC
# Checks for libraries.
# FIXME: Replace `main' with a function in `-lpthread':
AC_CHECK_LIB([pthread], [main])
# Checks for header files.
AC_HEADER_STDC
AC_CHECK_HEADERS([arpa/inet.h netinet/in.h string.h sys/socket.h unistd.h])
# Checks for typedefs, structures, and compiler characteristics.
AC_HEADER_STDBOOL
AC_C_CONST
AC_TYPE_PID_T
AC_TYPE_SIZE_T
AC_HEADER_TIME
# Checks for library functions.
AC_FUNC_SELECT_ARGTYPES
AC_CHECK_FUNCS([inet_ntoa memset select socket strstr])
#如果为多级目录树，所有的makefile文件需要在此列出。
AC_CONFIG_FILES([Makefile      src/Makefile           test/Makefile])  
AC_OUTPUT
#

###Makefile.am文件示例：
####根目录下的Makefile.am文件

SUBDIRS=src test 
KE_OPTIONS=foreign
AM_CPPFLAGS=-I ./src ./test

####src目录下的Makefile.am文件
bin_PROGRAMS=realview
realview_SOURCES=basetype.h console.cpp console.h http.cpp http.h ipdaemon.cpp ipdaemon.h \
MFCString.cpp MFCString.h
AM_LDFLAGS=-lpthread
AM_CPPFLAGS=-I ./src ../test

---
layout: post
title: "fix bug: timezone of toolchain"
date: 2011-01-22 00:00
comments: true
categories: [gcc]
tags: [toolchain, timezone, gcc]
---

When we compile a glibc(or eglibc), we need generated the timezone data file with it. although, it is stable and no change almost in every version update.

Today a problem is met about it.

We use the old glibc’s timezone file, which is used by many different toolchain for several paltforms. 

unfortunately.the data file has been change after 2007 year by GNU official. but I did not found the exact version(date) of glibc, which change the timezone data file.

btw: toolchain = binutils + gcc + glibc(eglic) + kernel(header)

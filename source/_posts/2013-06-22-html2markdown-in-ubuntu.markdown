---
layout: post
title: "Install html2markdown in Ubuntu(12.10)"
date: 2013-02-12 00:00
comments: true
categories: [others]
tags: [markdown, ubuntu, html2markdown, html2text]
---

<!-- more -->

###Edit source.list 
ensure the universe components is enabled.
martin@PC:~/git/blog/_posts$ cat /etc/apt/sources.list
deb http://ubuntu.cn99.com/ubuntu/ quantal main
deb-src http://ubuntu.cn99.com/ubuntu/ quantal main

deb http://ubuntu.cn99.com/ubuntu/ quantal universe
deb-src http://ubuntu.cn99.com/ubuntu/ quantal universe

###update apt
martin@PC:~/git/blog/_posts$ sudo apt-get update

###install
martin@PC:~/git/blog/_posts$ sudo apt-get install  python-html2text^C

### html2markdown ready
martin@PC:~/git/blog/_posts$ html2markdown --version
html2markdown 3.200.3



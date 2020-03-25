---
layout: post
title: "vim 特定字符后中插入回车换行"
date: 2007-09-14 00:00
comments: true
categories: [vim]
tags: [vim]
---
<!-- more -->
原文件  
`<port_countxsi:type=\"xsd:int\">16</port_count><switch_port_idxsi:nil=\"true\"/><attenuatorxsi:type=\"xsd:double\">+10</attenuator><ref_offsetxsi:type=\"xsd:int\">1</ref_offset><resol_bwdxsi:type=\"xsd:int\">300000</resol_bwd><video_bwdxsi:type=\"xsd:double\">+100000</video_bwd><cmd_port xsi:nil=\"true\"/><statusxsi:type=\"xsd:int\">13</status><pidxsi:nil=\"true\"/><firmware_refxsi:type=\"xsd:string\"/> <progress xsi:type=\"xsd:string\"/>`
这是一个xml文件的一部分，需要在“><”的中间插入回车换行，以方便浏览。

####命令
`%s/></>^M</g`
其中^M是ctrl+v加上ctrl+M的表示。
即按下ctrl 再按下v再按下m
####或者
`%s/></>\r</g`


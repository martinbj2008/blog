---
layout: post
title: "嵌入Linux下的usb storage的支持"
date: 2007-12-11 00:00
comments: true
categories: [others]
tags: [usb, kernel]
---

目标： 在mipsel架构的嵌入式linux系统上支持USB盘的读写。
思路： 使用可加载模块的形式增加 scsimod.ko, sdmod.ko, usbstorage.ko, fat.ko, vfat.ko.

##具体实现：
###步骤I 准备必须的驱动（可加载模块）
1. scsimod.ko         源代码目录下的 driver/scsi目录下的文件编译
2. sdmod.ko,          源代码目录下的 driver/scsi目录下的文件编译
3. usbstorage.ko   源代码目录下的 driver/usb/storage 目录下的文件编译
4. fat.ko               源代码目录下的 fs/fat目录下的文件编译
5. vfat.ko    源代码目录下的 fs/vfat目录下的文件编译
上述五个可加载模块的编译过程可以具体参考各自目录下的Makefile。

<!-- more -->

####我在编译过程中所犯的错误和经验：
    a. 没有真正明白     obj-m :=fat.o 和 fat-objs := 的关系， 直接将源文件`*.o`增加到了obj-m 的后面，造成没有编译成功。
    b. 在编译fat.ko的时候，inode。c文件里面的有个变量定义使用的配置文件（config）里面缺省值。
            `static int fat_default_codepage = CONFIG_FAT_DEFAULT_CODEPAGE;`
        我误以为是页面大小设置为4k， 导致在mount的时候死活mount文件系统失败。
        需要正确配置该变量的初始化值。
          在编译过内核的源代码的目录下有个隐含文件`.config`,  该文件里面有宏`CONFIG_FAT_DEFAULT_CODEPAGE`的定义。
	  `(CONFIG_FAT_DEFAULT_CODEPAGE=437)`

    c. 善于使用proc调试信息和dmesg查看最近的错误信息。 几个有帮助的proc文件。
     cat /proc/bus/usb/devices    所有识别的usb设备信息。如厂商号，设备名称。
                   本机设备的分区情况。可以看到U盘的分区。
     dmesg                                  最近的系统提示信息。
     fdisk -l /dev/sda                   显示sda设备的分区情况其分区类型。

###步骤II 加载模块，然后插入设备并手工创建设备节点并mount到特定目录。
        a. 依次加载上述5个模块。
        b.   插入usb设备
        c.    cat /proc/partitions得到设备的主次设备号。
        d.    mknod   /dev/sda    b major 0
              mknod   /dev/sda1 b major 1
             有可能根据具体设备配置情况，sd? 可能为sda ，或sdb 等。如果分区有多个要为每个分区都创建一个设备节点。
        e.    手动mount
               mount /dev/sda1 /usr/local/mnt
       至此 U盘可以正常读写了。在拔除U盘之前需要 umount /dev/sda1

##TODO
a. hotplug的支持，实现自动mount、umount及设备节点的管理  
b. 对NTFS分区的支持  
c. 图形操作界面及相应应用程序的自动挂载。  


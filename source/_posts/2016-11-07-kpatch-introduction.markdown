---
layout: post
title: "How to use kpatch"
date: 2016-11-07 16:20:10 +0800
comments: true
categories: []
---

### 搭建kpatch builder
以centos7.2为例。

默认centos7.2的安装的内核版本是`3.10.0-327.el7.x86_64`,这个内核版本当初是通过`gcc 4.8.3`编译的。
而centos7.2自带gcc rpm包的版本则是 `4.8.5`

kpatch build命令执行的时候，首先检查gcc的版本是否一致，
因为两者的版本不一致，所以kpatch build命令会失败。
当然我们可以使用`--skip-gcc-check`，跳过这个检查，我也测试发现在一些简单补丁下可以打包通过。
但是系统不推荐这样做的，会有一定的风险。

<!-- more -->

#### 搭建步骤:

+ 升级kernel

升级kernel版本到`kernel-3.10.0-327.36.3.el7.x86_64`
```
yum update kernel -y
kernel-3.10.0-327.36.3.el7.x86_64
```

+ 重启系统，并切换到新安装的内核

    reboot(switch to new kernel)

+ 安装所需的rpm包

```
yum install -y gcc kernel-devel elfutils elfutils-devel  rpmdevtools pesign yum-utils zlib-devel \
  binutils-devel newt-devel python-devel perl-ExtUtils-Embed \
  audit-libs audit-libs-devel numactl-devel pciutils-devel bison 
```

+ 打开debug源

```
 yum-config-manager --enable debug
```

+ 安装编译kernel所需的rpm报文

```
yum-builddep kernel
```

+ 安装debuginfo相关的rpm包

```
sudo debuginfo-install kernel
```

但是这个命令在选择kernel版本的时候，失败。
通过手动安装两个debuginfo的rpm报文，跳过了这一步

```
 rpm -ivh kernel-debuginfo-common-x86_64-3.10.0-327.36.3.el7.x86_64.rpm
 rpm -ivh kernel-debuginfo-3.10.0-327.36.3.el7.x86_64.rpm
```

+ 从github上下载并安装kpatch

centos默认的kpatch rpm里只有kpatch的命令行工具，
没有`kpatch-core.ko`, 这个ko是kpatch的必备文件。
kpatch的版本是`tag: v0.3.4`

```
yum install git -y
git clone https://github.com/dynup/kpatch.git
cd kpatch
make
make install(root)
```

### 制作热升级补丁
制作补丁很简单可以通过命令`kpatch-build`。
如：

```
kpatch-build vxlan.patch
```
第一次安装的时候，需要下载kernel src rpm包，速度会慢一些。
```
[root@vm1 ~]# kpatch-build  vxlan.patch
Fedora/Red Hat distribution detected
Downloading kernel source for 3.10.0-327.36.3.el7.x86_64
。。。
dev.o: changed function: __dev_set_promiscuity
dev.o: changed function: __dev_set_allmulti
dev.o: changed function: dev_set_mtu
dev.o: changed function: dev_queue_xmit
dev.o: changed function: unregister_netdevice_queue
dev.o: changed function: netdev_rx_handler_unregister
dev.o: changed function: validate_xmit_skb
dev.o: changed function: __dev_change_flags
dev.o: changed function: netdev_upper_dev_unlink
dev.o: new function: __kpatch_validate_xmit_skb
dev.o: changed function: netdev_has_upper_dev
dev.o: changed function: netdev_master_upper_dev_get
dev.o: changed function: netdev_has_any_upper_dev
dev.o: changed function: dev_change_net_namespace
dev.o: changed function: netdev_rx_handler_register
dev.o: changed function: __netdev_update_features
dev.o: changed function: register_netdevice
Patched objects: vmlinux
Building patch module: kpatch-vxlan.ko
SUCCESS
```
最后，在命令执行的当前目录下回产生一个ko文件`kpatch-vxlan.ko`

## 如何部署热升级补丁

### 服务器首次部署kpatch
#### 升级内核版本
确保被升级机器的内核版本跟builder机一致。

如不一致, 升级kernel到`3.10.0-327.36.3.el7.x86_64`

```
yum update kernel
```

#### 安装kaptch及kpatch core模块

```
yum install kpatch
insmod kpatch-core.ko
```
centos7.2自带的kpatch rpm包里只有kpatch的一些脚本命令，缺少`kpatch-core.ko`,
这个需要我们手动从builder机里copy。

### 安装kpatch 热补丁

#### 加载热补丁
```
kpatch install kpatch-vxlan.ko
```

#### 显示已安装的补丁
```
[root@vm2 ~]# kpatch list
Loaded patch modules:

Installed patch modules:
kpatch_vxlan (3.10.0-327.36.3.el7.x86_64)
```

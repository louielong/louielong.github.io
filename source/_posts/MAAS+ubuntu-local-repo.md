---
title: MAAS+ubuntu私有源环境搭建
date: 2018-03-27 13:56:18
tags:
  - MAAS
  - ubuntu
keywords:
  - MAAS
categories:
  - ubuntu
description:
  - MAAS安装使用以及ubuntu私有源环境搭建
summary_img:
---

## 前言

最近做一个集成项目需要部署多台裸机服务器，考虑到服务器众多采用PXE的方式给服务器安装系统是最便捷的同时考虑到网络速度的因素，建立一个ubuntu私有源镜像服务器安装软件包无疑会更快更便捷。

本文将分两部分讲解环境搭建：1)MAAS环境搭建，2)ubuntu私有源环境搭建。首先是MAAS环境的搭建。

## 1 MAAS环境搭建

### 1.1 MAAS介绍

![](https://assets.ubuntu.com/v1/33b96a7f-maas-logo.svg)

MAAS[1]是ubuntu社区开发的开源裸机部署工具，能够从云端下载镜像引导本地主机从PXE启动并安装指定的系统，搭配诸如IPAM等同时兼具裸机状态管理功能。

### 1.2 MAAS安装

```shell
root@ubuntu:~# apt install maas -y
```

### 1.3 创建MAAS用户

```shell
root@ubuntu:~# maas createadmin --username=admin --email=MYEMAIL@EXAMPLE.COM
```

随后登录MAAS web页面 `http://<your.maas.ip>/MAAS/`，然后配置相关参数

- Region name (MAAS name)
- Ubuntu archive, Ubuntu extra architectures
- Ubuntu images
- SSH keys (for currently logged in user)

![MAAS界面](https://i.imgur.com/mB3EK4K.jpg)

### 1.4 系统Images设置

在`Images`界面设置需要安装的系统版本，本次部署需要是64位的ubuntu 16.04，配置完成后MAAS会自动同步镜像

![MAAS Inages设置](https://i.imgur.com/tXMsQWz.jpg)

在`setting`界面配置系统PXE启动时装载的最小镜像供MAAS进行服务器的硬件信息等解析

![MAAS 最小镜像配置](https://i.imgur.com/gcsXtgW.jpg)

### 1.4  DHCP配置

点击`Subnets`选择一个网络点击`VLAN`选择右上的`take action`选项框，选择`Provid DHCP`，然后配置dhcp范围

![MAAS DHCP配置](https://i.imgur.com/0pPmBOZ.jpg)

【Note】

1)检查需要安装系统的服务器PXE启动，配置服务器启动项中PXE为首选；

2)配置服务器的IPMI，便于MAAS完全接管服务器，包括服务器的启动、关机、重启等。

启动待装系统的服务器，可以看到MAAS检测到服务器上线

![MAAS硬件检测](https://i.imgur.com/C2nozbq.jpg)

更多MAAS配置及设置参见官网[用户手册](https://docs.maas.io/2.3/en/)

## 2 ubuntu私有源环境搭建

私有源环境搭建主要是应对公司网络限制，如访问过慢或限制网络连接的情况，也可以应对需要大量现场装机部署等情况加快部署进度。

2.1 安装apt-mirror工具

```shell
root@ubuntu:~# apt-get install -y apt-mirror 
```

2.2 配置apt-mirror

本次安装的ubuntu为16.04，设置ubuntu源为清华源，同时可以修改源下载路径位置等参数。

```shell
root@ubuntu:~# cat /etc/apt/mirror.list 
############# config ##################
#
# set base_path    /var/spool/apt-mirror
#
# set mirror_path  $base_path/mirror
# set skel_path    $base_path/skel
# set var_path     $base_path/var
# set cleanscript $var_path/clean.sh
# set defaultarch  <running host architecture>
# set postmirror_script $var_path/postmirror.sh
# set run_postmirror 0
set nthreads     20
set _tilde 0
#
############# end config ##############
# louie add qinghua source 20180326
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse	

clean http://mirrors.tuna.tsinghua.edu.cn/ubuntu
```

随后运行apt-mirror下载源

```shell
root@ubuntu:~# apt-mirror 
Downloading 152 index files using 20 threads...
Begin time: Mon Mar 26 10:38:17 2018
[20]... [19]... [18]... [17]... [16]... [15]... [14]... [13]... [12]... [11]... [10]... [9]... [8]... [7]... [6]... [5]... [4]... [3]... [2]... [1]... [0]... 
End time: Mon Mar 26 10:38:29 2018

Processing tranlation indexes: [TTTT]
.......
```

随后等待文件下载完成，等待时间视网络状况，本次源仓库共有113.3 GiB。(ps：在更新了三个小时多后终于更新完成了)

***【Note】***

1)当某些软件包在服务器端进行了升级，或者服务器端不再需要这些软件包时，我们使用了 `apt-mirror`与服务器同步后，会在本地的`$var_path/`下生成一个`clean.sh`的脚本，列出了遗留在本地的旧版本和无用的软件包，可以手动运行这个脚本来删除遗留在本地的且不需要用的软件包
`clean http://mirrors.tuna.tsinghua.edu.cn/ubuntu`

2)如果用amd64位架构下的包，可以加上deb-amd64的标记如果什么都不加，直接使用deb http.....这种格式，则在同步时，只同步当前系统所使用的架构下的软件包。比如一个64位系统，直接deb http....只同步64位的软件 包。如果还嫌麻烦，直接去改`set defaultarch   <running hostarchitecture>`这个参数就好，比如改成set defaultarch i386，这样你使用debhttp.....这种格式，则在同步时，只同步i386的软件包了。

如果你还想要源码，可以把源码也加到mirror.list里面同步过来，比如加上deb-src这样的标记。想要其他的东西也可以追加相应的标记来完成。

3）同步完成后，我们可以利用clean.sh清理无用软件包：

```shell
 root@ubuntu:~# /var/spool/apt-mirror/var/clean.sh 
```

2.3 作为本机源配置

源路径为

配置本机源文件

```shell
 root@ubuntu:~# cat /etc/apt/sources.list
 deb file:///var/spool/apt-mirror/mirror/mirrors.tuna.tsinghua.edu.cn/ubuntu trusty main restricted universe multiverse
deb file:///var/spool/apt-mirror/mirror/mirrors.tuna.tsinghua.edu.cn/ubuntu trusty-security main restricted universe multiverse
deb file:///var/spool/apt-mirror/mirror/mirrors.tuna.tsinghua.edu.cn/ubuntu trusty-updates main restricted universe multiverse
deb file:///var/spool/apt-mirror/mirror/mirrors.tuna.tsinghua.edu.cn/ubuntu trusty-proposed main restricted universe multiverse
deb file:///var/spool/apt-mirror/mirror/mirrors.tuna.tsinghua.edu.cn/ubuntu trusty-backports main restricted universe multiverse 
```

2.4 局域网源配置

2.4.1 首先需要安装apache2服务器

```shell
 root@ubuntu:~# apt-get install apache2 
```

将镜像目录链接到apache2的根目录(/var/www/html/)下 

```shell
 root@ubuntu:~# sudo ln -s /var/spool/apt-mirror/mirror/mirrors.tuna.tsinghua.edu.cn/ubuntu /var/www/html/ubuntu
```

打开浏览器`http://<HOST IP>/ubuntu` 即可查看

2.4.2 修改局域网ubuntu

```shell
 root@ubuntu:~# cat /etc/apt/sources.list
deb http://192.168.4.170/ubuntu trusty main restricted universe multiverse  
deb http://192.168.4.170/ubuntu trusty-security main restricted universe multiverse  
deb http://192.168.4.170/ubuntu trusty-updates main restricted universe multiverse  
deb http://192.168.4.170/ubuntu trusty-proposed main restricted universe multiverse  
deb http://192.168.4.170/ubuntu trusty-backports main restricted universe multiverse
```



【参考链接】

1)[MAAS安装指导](https://maas.io/install)

2)[ubuntu私有源搭建](https://blog.csdn.net/wenwenxiong/article/details/50908002)

3)[私有源配置](https://www.linuxidc.com/Linux/2014-08/105415.htm)
































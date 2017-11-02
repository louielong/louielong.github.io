---
title: OPNFV Euphrates 安装（一）
date: 2017-10-31 15:05:57
tags:
  - OPNFV
  - Euphrates
keywords:
  - Euphrates
categories:
  - OPNFV
description:
  - OPNFV Euphrates
summary_img:
  - https://i.imgur.com/yNc6FFv.jpg
top: 100
---

## 1 Euphrates新特性介绍

OPNFV社区于2017年10月24发布的第五个版本Euphrates（幼发拉底河，位于亚洲），新版本增加了对***Kubernetes***容器技术的支持，也就是说OPNFV的底层VIM不再仅仅是Openstack架构了，同时社区采用了***XCI***（cross community continuous integration）交叉持续集成技术，能够更快更好的使用上游社区最新的版本和代码，在MANO管理层集成了ETSI NFV的***Open Baton***项目。E版本的详细新特性如下[1][2]：

- 底层架构增加：OPNFV即将开启容器之旅。Euphrates将Kubernetes和容器集成到端到端堆栈的多个组件，以及通过Kolla部署容器化OpenStack的能力，Kolla提供生产就绪的容器和部署工具，用于运行可扩展、快速、可靠的OpenStack云，并使用社区最佳实践升级。这些增强功能可以更轻松地管理基础设施、支持NFV中的原生云网络应用，以及轻权重控制平面功能，为运营商支持5G和IoT奠定基础。
- XCI持续集成：能够与最新的上游开源项目集成。在OPNFV第四个版本Danube中持续集成/持续部署（CI/CD）集成工作的基础上，Euphrates在OPNFV、OpenStack、OpenDaylight和FD.io中引入了XCI集成CI/CD管道的实现。OPNFVC CI管道并不需要官方的稳定版本，而是集成了上游项目的最新代码，以更快地解决错误并验证功能。这减少了新功能反馈和错误修复的时间，大大提升了创新的速度。XCI还可实现多分布式支持，促进开发人员之间的联系。
- 电信级功能。通过整合Calipso项目，运营商现在可以对其复杂的虚拟网络进行可视化操作。结合现有的Barometer and Doctor项目中的遥测增强功能，用户可以获得强大的服务保障架构。Euphrates还包括ARM架构的性能改进，以及FD.io的3层性能。此外，Euphrates通过Moon给用户带来新的安全的管理功能，持续改进了服务功能链（SFC）、FD.io、EVPN性能。Euphrates还将OVN网络虚拟化项目以及最新版本的其他上游项目整合在一起，为网络控制提供额外的选择。
- 测试和集成功能增强：OPNFV集成和测试工作提供广泛的工具来测试NFV云、VNF和完整的网络服务方面取得了重大进展。包括Sample VNF在内的新项目提供对VIM/NFVI层的测试，以及接近实际应用工作负载的应用程序。NFVBench项目提供了端到端数据平台基准测试架构。

## 2 Euphrates 安装

### 2.1 配置要求

官方对于虚拟部署的配置要求如下

| 硬件           | 配置需求                               |
| ------------ | ---------------------------------- |
| 1 Jumpserver | 1个物理的节点用于安装Salt Master虚拟机和其他节点的虚拟机 |
| CPU          | 最少一个插槽并且支持虚拟化                      |
| RAM          | 最小32G内存（与VNF的工作负载有关）               |
| Disk         | 最小100G（SSD或者HDD（15krpm最佳））         |

本文采用fuel部署脚本在一个服务器上部署虚拟POD，硬件配置及系统如下

- Dell R720服务器，8核E5-2609，24G内存
- ubuntu 16.06 server，8 cpu，24G内存，300G磁盘

本文中将称ubuntu 16.04系统为host。

### 2.1 部署前准备

### 2.1.1 CPU开启KVM虚拟化

本次安装使用的是在ESXI上安装的虚拟机，需要开启CPU的KVM虚拟化支持

![](https://i.imgur.com/Xo9eSSj.jpg)

### 2.1.2 虚拟机管理工具virsh

```shell
sudo apt-get -y install libvirt-bin
```

*可选步骤*：修改host上软件源以加快部署进度，将软件源替换为[清华源](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/ " 清华源")

修改`/etc/apt/source.list`文件内容全部替换为

```shell
  # 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
  deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse
  deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
  deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
  deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
  # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted universe multiverse	
```

替换完成之后运行`sudo apt-get update`

### 2.2 fuel脚本部署Euphrates

下载x86版本fuel部署脚本并切换至5.02版本分支[3]

```shell
git clone https://git.opnfv.org/fuel
git checkout opnfv-5.0.2
```

*Note*：最开始尝试部署时由于内存只有8G出现过未部署成功，因此需要适当削减部署时的配置，修改`fuel/mcp/config/scenario/defaults-x86_64.yaml`中ram的大小，以及`mcp/config/scenario/virtual/os-nosdn-nofeature-noha.yaml`中ram的大小，甚至可以减少一个计算节点`cmp02`。

最简单的虚拟部署方式命令如下

```shell
ci/deploy.sh -d -p virtual_kvm -l localhost -s os-nosdn-nofeature-noha
```

使用`deploy -h`可以常看使用帮助，常用的参数说明如下

| -b   | 指定部署配置的目录可以使用本地文件和远程链接，配置文件的路径格式为`<base-uri>/labs/<lab-name>/<pod-name>.yaml`,默认路径为`./mcp/config` |
| ---- | ---------------------------------------- |
| -l   | 指明使用配置目录下哪个实验室的配置文件，如`-l lf`为使用linux基金会实验室的配置文件。当使用虚拟部署时`-l`参数可以不使用 |
| -p   | 指明使用实验室下的哪一个POD，如`-p pod2`               |
| -s   | 指明使用哪种部署场景，如`-s os-nosdn-nofeature-noha`指使用openstack作为VIM，不采用sdn控制器，无其他特性，不使用ha。其他场景可以在`fuel/mcp/config/scenario/`下看到 |

使用ssh登录到生成的虚拟机中

```shell
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /var/lib/opnfv/mcp.rsa ubuntu@10.20.0.2
```

本次部署过程中并未指定节点的相关信息因此诸如控制节点、计算节点等信息是随机生成，在接下来的第二篇详细介绍。



**参考链接**：

1)[OPNFV官方WIKI](https://www.opnfv.org/software)

2)[OPNFV 第五个版本发布](http://www.sdnlab.com/20009.html)

3)[Euphrates安装手册](http://docs.opnfv.org/en/stable-euphrates/submodules/fuel/docs/release/installation/installation.instruction.html)










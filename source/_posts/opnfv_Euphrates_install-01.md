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
  - https://raw.githubusercontent.com/louielong/blogPic/master/imgyNc6FFv.jpg
top:
---

## 1 Euphrates新特性介绍

OPNFV社区于2017年10月24发布的第五个版本Euphrates（幼发拉底河，位于亚洲），新版本增加了对***Kubernetes***容器技术的支持，也就是说OPNFV的底层VIM不再仅仅是Openstack架构了，同时社区采用了***[XCI](http://docs.opnfv.org/en/latest/submodules/releng-xci/docs/xci-overview.html#xci-overview)***（cross community continuous integration）跨社区持续集成技术，能够更快更好的使用上游社区最新的版本和代码，在MANO管理层集成了ETSI NFV的***Open Baton***项目。E版本的详细新特性如下[1][2]：

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
| Disk         | 最小100G（SSD或者HDD（15k rpm最佳））        |

本文采用fuel部署脚本在一个服务器上部署虚拟POD，硬件配置及系统如下

- Dell R720服务器，8核E5-2609，24G内存
- ubuntu 16.06 server，8 cpu，24G内存，300G磁盘

本文中将称ubuntu 16.04系统为host。

### 2.1 部署前准备

#### 2.1.1 CPU开启KVM虚拟化

本次安装使用的是在ESXI上安装的虚拟机，需要开启CPU的KVM虚拟化支持

![](![http://vps.ylong.co:8686/file/Xo9eSSj.jpg](http://vps.ylong.co:8686/file/Xo9eSSj.jpg))

#### 2.1.2 虚拟机管理工具virsh

```shell
sudo apt-get -y install libvirt-bin
```

*2017年12月12日更新*：社区最新的代码中已经添加了libvirt的安装，将安装过程调整为先安装libvirt后做`virsh list`的检查，因此该步骤安装可以省略。

*可选步骤*：在国内为了加快部署进度，可以修改host上软件源将软件源替换为[清华源](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/ " 清华源")

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

*2017年12月12日更新*：由于XCI的特性，社区的代码实时更新，使用`git fetch origin stable/euphrates && git checkout stable/euphrates`切换至Euphrates的最新稳定版代码，然后再部署。

*Note*：最开始尝试部署时由于内存只有8G出现过未部署成功，因此需要适当削减部署时的配置，修改`fuel/mcp/config/scenario/defaults-x86_64.yaml`中ram的大小，以及`mcp/config/scenario/virtual/os-nosdn-nofeature-noha.yaml`中ram的大小，甚至可以减少一个计算节点`cmp02`。

最简单的虚拟部署方式命令如下

```shell
ci/deploy.sh -D -p virtual_kvm -l localhost -s os-nosdn-nofeature-noha
```

使用`deploy -h`可以常看使用帮助，常用的参数说明如下

| -b   | 指定部署配置的目录可以使用本地文件和远程链接，配置文件的路径格式为`<base-uri>/labs/<lab-name>/<pod-name>.yaml`,默认路径为`./mcp/config` |
| ---- | ---------------------------------------- |
| -l   | 指明使用配置目录下哪个实验室的配置文件，如`-l lf`为使用linux基金会实验室的配置文件。当使用虚拟部署时甚至`-l`参数都可以不使用 |
| -p   | 指明使用实验室下的哪一个POD，如`-p pod2`               |
| -s   | 指明使用哪种部署场景，如`-s os-nosdn-nofeature-noha`指使用openstack作为VIM，不采用sdn控制器，无其他特性，不使用ha。其他场景可以在`fuel/mcp/config/scenario/`下看到 |

使用ssh登录到生成的虚拟机中,登录虚拟机后使用`sudo su`命令即可切换成root用户。

```shell
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /var/lib/opnfv/mcp.rsa ubuntu@10.20.0.2
```

本次部署过程中并未指定节点的相关信息因此诸如控制节点、计算节点等信息是随机生成，在接下来的第二篇详细介绍。

### 2.3 dashboard的访问

由于部署的是虚拟环境，各网络都是内部网络（internal network）因此需要做路由转换才能够正常访问。

首先需要找到控制节点的IP通过登录到配置节点cfg01查找其他节点，使用`arp -a`命令可以看到其他节点的IP。

![其它节点IP](https://raw.githubusercontent.com/louielong/blogPic/master/imgt09x2yW.jpg)

登录到ctl01节点上查看文件`/etc/apache2/conf-available/openstack-dashboard.conf`可以得知dashboard页面的监听端口为8078。

访问dashboard的方式有**两种**，第一种是在host上做端口映射；第二种是通过putty、xshell等终端登录工具做隧道。不管哪种方式由于fuel在部署时防火墙的配置中多了一些选项需要先对iptables做一些修改。

首先在host上备份防火墙规则`iptables-save > rule.v4`,然后打开rule.v4修改`-A FORWARD -d 10.20.0.0/24 -o mcpcontrol -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT`为`-A FORWARD -d 10.20.0.0/24 -o mcpcontrol -j ACCEPT`，随后重新加载防火墙规则`iptables-restore rule.v4`。

#### 2.3.1 隧道方式访问

打开终端的配置页面增加隧道条目，如图所示将本机的8000端口映射到控制节点的8078端口

![隧道配置](https://raw.githubusercontent.com/louielong/blogPic/master/imgD1CDnUI.jpg)

随后在本机的浏览器输入`127.0.0.1:8000`即可访问dashboard，**账户/密码：admin/opnfv_secret**。密码存放在控制节点的`/etc/keystone/keystone.conf`文件中，为`admin_token`参数的值。

#### 2.3.2 host端口映射方式访问

在host上的防火墙规则中添加一个端口映射，`iptables -t nat -A PREROUTING -d 192.168.2.235 -p tcp --dport 8000 -j DNAT --to-destination 10.20.0.118:8078`，随后直接访问host主机的8000端口即可登录dashboard。



**参考链接**：

1)[OPNFV官方WIKI](https://www.opnfv.org/software)

2)[OPNFV 第五个版本发布](http://www.sdnlab.com/20009.html)

3)[Euphrates安装手册](http://docs.opnfv.org/en/stable-euphrates/submodules/fuel/docs/release/installation/installation.instruction.html)










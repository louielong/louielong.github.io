---
title: Dovetail 项目说明
date: 2017-09-27 17:10:13
categories:
  - OPNFV
tags:
  - Dovetail
  - OPNFV
  - NFV
keywords:
  - Dovetail
description:
  - OPNFV Dovetail 使用
---

***NOTE***：本文以dovetail CVP 的0.6版本为例来讲解dovetail项目的使用，随着dovetail版本的升级使用的命令和配置会有所改变。（2017年9月8日13:37:05）

**摘要**:本文主要介绍如何使用OPNFV的Dovetail项目进行NFV基础设施NFVI&VIM的测试。

<!--more-->

## 1.Dovetail 安装

dovetail 可以运行在实体机(bare metal)，可以运行在虚拟机中。dovetail可以使用源码来安装也可以使用docker环境，建议使用docker来安装。为了环境部署方便以及更快的熟悉该项目，本文采用 docker 环境来安装和使用 dovetail 。

### 1.1 准备相应的测试代码

由于dovetail测试项目需要使用OPNFV中的其他测试项目代码，因此还需要functest、yardstick等项目。

```shell
docker pull opnfv/dovetail:cvp.0.6.0
docker pull opnfv/functest:cvp.0.5.0
docker pull opnfv/yardstick:danube.3.2
docker pull opnfv/bottlenecks:cvp.0.4.0
wget -nc http://artifacts.opnfv.org/sdnvpn/ubuntu-16.04-server-cloudimg-amd64-disk1.img -P .
wget -nc http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img -P .
```

dovetail可以在离线环境下进行测试，前提是需要先保存以上的docker镜像。

```shell
 docker save -o dovetail.tar opnfv/dovetail:cvp.0.6.0 opnfv/functest:cvp.0.5.0 \
 opnfv/yardstick:danube.3.2 opnfv/bottlenecks:cvp.0.4.0
```

### 1.2 配置 dovetail 运行环境

首先创建一个dovetail的相关文件存放的目录`${DOVETAIL_HOME}/pre_config`，这里以`/root/dovetail`为例

```shell
export DOVETAIL_HOME=/home/dovetail
```

将之前的镜像文件保存在`pre_config`目录下，并在该目录下创建`env_config.sh`文件，并填入一下内容。需要指出的是dovetail CVP中不在需要`INSTALL_IP`之类的参数，相关警告可以忽略，详见

https://jira.opnfv.org/browse/DOVETAIL-371

```shell
# Project-level authentication scope (name or ID), recommend admin project.
export OS_PROJECT_NAME=admin
# Authentication username, belongs to the project above, recommend admin user.
export OS_USERNAME=admin
# Authentication password. Use your own password
export OS_PASSWORD=admin
# Authentication URL, one of the endpoints of keystone service. If this is v3 version,
# there need some extra variables as follows.
export OS_AUTH_URL='http://192.168.20.11:5000/'
# Default is 2.0. If use keystone v3 API, this should be set as 3.
export OS_IDENTITY_API_VERSION=3
# Domain name or ID containing the user above.
# Command to check the domain: openstack user show <OS_USERNAME>
export OS_USER_DOMAIN_NAME=default
# Domain name or ID containing the project above.
# Command to check the domain: openstack project show <OS_PROJECT_NAME>
export OS_PROJECT_DOMAIN_NAME=default
# Home directory for dovetail
export DOVETAIL_HOME=/root/dovetail
```

导入该环境变量

```shell
source env_config.sh
```

### 1.3 运行 dovetail 容器

```shell
docker run --privileged=true -it \
             -e DOVETAIL_HOME=$DOVETAIL_HOME \
             -v $DOVETAIL_HOME:$DOVETAIL_HOME \
             -v /var/run/docker.sock:/var/run/docker.sock \
             --name dovetail opnfv/dovetail:cvp.0.6.0 /bin/bash 
```

上述命令中`-v $DOVETAIL_HOME:$DOVETAIL_HOME`是指明host与container之间目录映射，此处不一致会导致后续的测试结果存储出现问题。在后续拍错时可以使用`docker inspect dovetail`命令检查容器的相关配置。运行完成后会直接进入到dovetail容器中.

为了方便的看清目录和文件，增加一些配置

```shell
alias l='ls -CF'
alias la='ls -A'
alias ll='ls -alF'
alias ls='ls --color=auto'
```

## 2. 执行测试

### 2.1 简单用例测试

使用命令查看有哪些dovetail测试例

```shell
dovetail list
```

运行 dovetail 测试

当dovetail正式版本发布时使用下列命令执行整个测试

```shell
dovetail run --testsuite CVP_1_0_0 --testarea ipv6 --debug | tee test.txt
```

目前dovetail测试版本使用下列命令执行测试

```shell
dovetail run -d --testsuite proposed_tests --testarea ha --offline | tee $DOVETAIL_HOME/test.txt-20170912_0954-HA
```

`testarea`指明测试域，在dovetail容器的`conf`目录中`dovetail.yml`可以查看相关测试域

> testarea_supported:
>
> - defcore
> - example
> - ha
> - ipv6
> - sdnvpn
> - vping
> - resiliency
> - tempest

或者使用`dovetail list proposed_tests `命令来查看有哪些`testarea`和具体的`testarea`下包含的测试例是什么。

### 2.2 其他测试配置文件

### 2.2.1 HA测试配置

HA测试的配置文件为`${DOVETAIL_HOME}/pre_config/pod.yaml`，可以参考`dovetail/userconfig/pod.yaml.sample`配置内容如下,只需要列出HA的控制节点

```
nodes:
-
    # This can not be changed and must be node1.
    name: node1

    # This must be controller.
    role: Controller

    # This is the install IP of a controller node.
    ip: xx.xx.xx.xx

    # User name of this node. This user must have sudo privileges.
    user: root

    # Password of the user.
    password: root
    
    # Private key of this node. It must be /root/.ssh/id_rsa
    # Dovetail will move the key file from $DOVETAIL_HOME/pre_config/id_rsa
    # to /root/.ssh/id_rsa of Yardstick container
    key_filename: /root/.ssh/id_rsa
```

需要注意的是节点的名字必须是`node1`、`node2`格式，主要是该项测试调用了functest的相关测试，而functest做了相关配置要求，IP地址为**测试机**能够访问节点的IP，HA测试会需要ssh登录到节点中关闭/重启openstack的nova、glance等相应服务，也需要提供能够访问其他节点的密匙文件。

#### 2.2.2 tempest测试配置

配置文件需要存放在`$DOVETAIL_HOME/pre_config/tempest_conf.yaml`,依据待测环境配置相应的信息。

```shell
validation:
    image_ssh_user: cirros
    ssh_timeout: 300

compute:
    min_compute_nodes: 2
#    volume_device_name: vdb
```

测试过程中使用的是cirros的镜像，cirros的初始账户为：cirros，设置登录超时时间，设置计算节点个数，对于compass的环境新添加的系统云盘名称为vdb因此需要指定（默认为vdc）。

## 3. 测试结果

dovetail的测试结果存放在`$DOVETAIL_HOME/results`目录下，具体的每个测试域也有对应的详细存储结果目录。如IPv6测试详细结果存放在`ipv6_logs/dovetail.ipv6.tcXXX.log`，可以查看执行失败的原因。



**参考链接**

1）[dovetail 用户手册](http://artifacts.opnfv.org/dovetail/docs/testing_user_userguide/index.html)


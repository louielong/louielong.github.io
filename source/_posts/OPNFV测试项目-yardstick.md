---
title: OPNFV 测试项目 -- Yardstick
date: 2017-06-09 15:20:08
tags:
  - Yardstick
  - OPNFV
  - Openstack
categories:
  - OPNFV
keywords:
  - Yardstick
description:
  - OPNFV Yardstick 使用
---

**摘要**:本文主要介绍如何使用OPNFV的Yardstick测试项目的使用。

<!--more-->

## 1.使用 Docker 安装 Yardstick
**官方推荐 Docker 运行 Yardstick 测试**

### 1.1安装 Docker

```shell
wget -qO- https://get.docker.com/ | sh
```

拉取 Yardstick 最新版

```shell
docker pull opnfv/yardstick:latest
```

查看下载的 Docker 镜像

```shell
docker images
```

## 2.准备 Yardstick 测试所需要的磁盘和镜像

*NOTE*:当需要使用 openstack 的接口时必须先导入环境变量

将下列配置将保存到 openrc 文本中

```shell
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.20.11:5000     #yardstick API接口的IP
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export EXTERNAL_NETWORK=admin_floating_net       #外网的网络名
```

使用 `source openrc`导入环境变量

```shell
source openrc
```



在 Yardstick 容器中，Yardstick 存储库位于 /home/opnfv/repos 目录中。Yardstick 提供了一个 CLI 来准备 OpenStack 环境变量并自动创建 Yardstick 风格和访客镜像：

### 2.1手动创建 Yardstick falvor 和 guest images

在执行 Yardstick 测试用例之前，请确保在 OpenStack 中提供了 Yardstick falvor 和 guest images。有关创建 Yardstick falvor 和 guest images 的详细步骤，如下。

使用如下命令创建一个名称为 yardstick-flavor ，编号为100，内存为512M，3G磁盘，1个cpu的主机配置。

```shell
nova flavor-create yardstick-flavor 100 512 3 1
```

|  参数  |    说明    |
| :--: | :------: |
| 100  | 创建镜像的ID  |
| 512  |   内存大小   |
|  3   | 磁盘大小 (G) |
|  1   |  CPU数量   |

![](https://raw.githubusercontent.com/louielong/blogPic/master/imgEhW1nuk.jpg)

### 2.2创建用于 Yardstick 测试的访客镜像

首先安装额外的工具包

```shell
sudo apt-get update && sudo apt-get install -y qemu-utils kpartx
```

使用 Git 拉取 Yardstick 源码

```shell
git clone https://gerrit.opnfv.org/gerrit/yardstick
```

使用 Yardstick 工具生成镜像

```shell
cd yardstick
sudo ./tools/yardstick-img-modify tools/ubuntu-server-cloudimg-modify.sh
```

生成的镜像也加入到 Openstack 环境中

```shell
glance --os-image-api-version 1 image-create \
--name yardstick-image --is-public true \
--disk-format qcow2 --container-format bare \
--file /tmp/workspace/yardstick/yardstick-image.img
```

一些 Yardstick 的测试用例使用 Cirros 0.3.5 或者 Ubuntu 16.04 的镜像，也可以将这些镜像加入到 Openstack 环境中

```shell
openstack image create \
--disk-format qcow2 \
--container-format bare \
--file $cirros_image_file \
cirros-0.3.5

openstack image create \
--disk-format qcow2 \
--container-format bare \
--file $ubuntu_image_file \
Ubuntu-16.04
```

## 3.运行 Yardstick

### 3.1启动 Yardstick 镜像

```shell
docker run -itd --privileged -v /var/run/docker.sock:/var/run/docker.sock -p 8888:5000 -e INSTALLER_IP=192.168.20.5 -e INSTALLER_TYPE=fuel --name yardstick opnfv/yardstick:latest
```

**参数说明**

|                    参数                    |                    说明                    |
| :--------------------------------------: | :--------------------------------------: |
|                   -itd                   | -i：交互式，即使不附加STDIN也会打开。 -t：分配伪TTY。 -d：在分离模式下，在后台运行容器。 |
|               –privileged                |          使容器内的root用户拥有真正的root权限          |
| -e INSTALLER_IP=192.168.20.5 <br /> -e INSTALLER_TYPE=fuel | INSTALLER_TYPE可选参数为Apex，Compass，Fuel和Joid。 如果使用其他安装程序（如devstack），则可以忽略这些参数。 |
|               -p 8888:5000               | 如果需要在yardstick外使用yardstick容器需要加入该参数，将yardstick容器内的5000端口映射到宿主机的8888端口，如果宿主机的8888端口被占用可以换一个端口 |
| -v /var/run/docker.sock:/var/run/docker.sock |                                          |
|             –name yardstick              |                 创建的容器名称                  |

### 3.2配置 Yardstick 容器环境

首先需要先进入到 Yardstick 容器

```shell
docker exec -it yardstick /bin/bash
```

随后开始配置环境

引入环境配置将一下内容保存到 openrc 文件中，然后使用 `source openrc`

```shell
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.20.11:5000     #yardstick API接口的IP
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export EXTERNAL_NETWORK=admin_floating_net       #外网的网络名
```

### 3.3运行一个简单的测试例

修改 ping.ymal文件

```shell
cd ~/repos/yardstick
vim samples/ping.yaml
```
![](https://raw.githubusercontent.com/louielong/blogPic/master/imgrvEEYJv.jpg)

主要关注下图中绿色框内容，*image* 为测试的镜像名称，*flavor* 为生成的虚拟机配置，*user* 为虚拟机系统的登录用户名(cirros 镜像默认的用户名和密码为"user:cirros  pass:cubswin:)"，ubuntu 镜像用户名默认为"ubuntu")，测试用的vm实例创建后会使用创建时的密匙登录，因此也是不需要密码的，但是系统用户名必须一致。*network* 指生成的虚拟机所在的网段，即内网的网络。
![ping 配置文件](https://raw.githubusercontent.com/louielong/blogPic/master/imgataU1pE.jpg)

配置完 `ping.ymal`文件后，使用命令启动测试

```shell
yardstick -d task start samples/ping.yaml
```

测试结果会输出在`/tmp/yardstick.out`中，为了方便的查看测试结果，使用 influxdb 数据库和 Grafana 工具。

## 4.安装 InfluxDB 和 Grafana

官方给出的 `yardstick env influxdb` 和 `yardstick env grafana` 使用版本较低，建议手动安装和启动这两项服务

完整的操作过程为:

### 4.1配置 InfluxDB 环境

使用 docker 安装并使用最新的 influxdb 版本

```shell
docker pull tutum/influxdb
```

启动 influxdb 的 docker 环境

```shell
docker run -d --name influxdb -p 8083:8083 -p 8086:8086 --expose 8090 --expose 8099 tutum/influxdb
docker exec -it influxdb bash  #进入 influxdb 的 docker 环境
```

配置 influxdb 数据库，增加 yardstick 表，便于后续存储 yardstick 测试结果

```shell
influx
>CREATE USER root WITH PASSWORD 'root' WITH ALL PRIVILEGES
>CREATE DATABASE yardstick;
>use yardstick;
>show MEASUREMENTS;
```

在 Yardstick 的docker容器中使用如下命令测试 influxdb 数据库是否可用

```shell
curl -i -X POST  http://192.168.20.14:8086/query --data-urlencode "q=CREATE DATABASE mydb"
```

`192.168.20.14`为 influxdb 运行所在的宿主机 IP 地址，上述命令的功能是使用 influxdb 的 http API 在数据库中创建一个数据库

**NOTE**如果上述命令执行失败，则需要查看宿主机的 防火墙规则，使用`iptables -L -nv`查看 `Chain INPUT`规则，如果是 drop 则使用`iptables -P INPUT ACCEPT`将其改为accept

### 4.2配置 Grafana 环境

拉取 Grafana 在 docker 上的最新版

```shell
docker pull grafana/grafana
```

```shell
docker run -d --name grafana -p 3000:3000 grafana/grafana
```
之后就可以在 http://host_ip:3000 (admin/admin) 查看

如下图所示为三个环境容器都已经启动，因8888端口冲突所以将 Yardstick 的宿主机端口改为45678
![](https://raw.githubusercontent.com/louielong/blogPic/master/imgOKtO9SO.jpg)

4.3配置 Yardstick 测试结果输出到 influxdb 数据库

进入到 Yardstick 容器的环境

```shell
docker exec -it yardstick bash
```

拷贝并修改 Yardstick 的配置

```shell
cp yardstick/etc/yardstick/yardstick.conf.sample /etc/yardstick/yardstick.conf
vim /etc/yardstick/yardstick.conf
```
![](https://raw.githubusercontent.com/louielong/blogPic/master/img13WSHjN.jpg)


修改*dispatcher*为*influxdb*，同时修改*influxdb*的IP地址，数据库名已经账号密码皆为4.1节所创建的。

配置完成后运行`yardstick -d task start samples/ping.yaml`

### 4.4在 Grafana 的web页面查看

在浏览器输入 http://192.168.20.14:3000 (admin/admin)(本例中的 Grafana 宿主机地址是 192.168.20.14)

点击左上角图标配置 data source，随后创建 dashboard 查看数据即可。

![](https://raw.githubusercontent.com/louielong/blogPic/master/imgteplLiT.jpg)



## 5.删除 Yardstick 容器

```shell
docker stop yardstick && docker rm yardstick
```



参考链接：

1)https://docs.opnfv.org/en/stable-danube/submodules/yardstick/docs/testing/user/userguide/index.html




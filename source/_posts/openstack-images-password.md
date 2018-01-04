---
title: openstack中镜像的密码修改
date: 2018-01-04 10:17:24
tags: 
   - Openstack
   - OPNFV
keywords:
   - image password
categories:
   - Openstack
description:
   - openstack 镜像密码修改
summary_img:
---

<span id="jump0">开始</span>

## 1 前言

本文主要讲解如何修改openstack中镜像的密码以及开启镜像的ssh登录。

openstack中的镜像登录方式主要有以下几种：

1）openstack的VNC终端密码登录；

2）ssh使用密匙登录；

3）ssh使用密码登录。

除了密匙登录其他两种都需要密码，而一般的镜像初始状态是不能使用密码登录或者说密码未知的，因此需要对镜像进行处理，处理方法有以下几种：

1）通过openstack的实例创建执行脚本修改；

2）通过直接修改镜像加入初始密码。

多数的系统镜像都加入了普通用户通过`sudo su`切换成root用户，原因是`/etc/sudoers`加入的`%sudo   ALL=(ALL:ALL) ALL`。

## 2 镜像处理介绍

### 2.1 cirros

cirros镜像是目前openstack中测试使用非常广泛的镜像，体积较小，易于使用，下载地址：https://download.cirros-cloud.net/

cirros 0.3.5的镜像账号密码为

```
user:cirros
pass:cubswin:)
```

cirros 0.4.0的账号密码为

```
user:cirros
password:gocubsgo
```

不同版本的cirros的镜像密码可能不同，但是在控制台日志中都会显示，同时该镜像也默认开启了ssh登录，可以使用账号密码登录。如无法登录记得查看镜像使用的安全组是否开始ssh访问权限

![openstak-SR-ssh](https://i.imgur.com/5P14D7V.jpg)

登录之后使用`sudo su`切换成root用户，若想直接使用root用户登录，需要拷贝密钥或者修改root用户密码，拷贝密钥的命令为：

```shell
cp -f /home/cirros/.ssh/authorized_keys /root/.ssh/
```



### 2.2 ubuntu镜像

ubuntu系统镜像的官方下载地址为：http://cloud-images.ubuntu.com

trusty为ubuntu 14，xenial为ubuntu 16，根据自己的喜好下载镜像。

### 2.2.1 修改镜像

使用guestfish工具直接修改镜像[1]，安装guestfish工具

```shell
sudo apt-get install libguestfs-tools -y
```

打开镜像：

```shell
sudo guestfish --rw -a xenial-server-cloudimg-amd64-disk1.img
```

挂载文件系统等操作如下图所示：

![guestfish change passwd](https://i.imgur.com/TVe8pr4.jpg)

打开`/etc/cloud/cloud.cfg`后修改一下内容：

1）增加ssh密码登录

将`disable_root`的值设为`false`即可允许root登录，增加`ssh_pwauth: true`即可允许ssh密码登录。

![openstack-passwd2](https://i.imgur.com/Rzj5T7u.jpg)

2）增加默认用户ubuntu的密码

将`lock_passwd`设为`false`允许VNC终端密码登录，同时添加`plain_text_passwd: "ubuntu"`将默认用户的密码设为`ubuntu`。

![openstack-passwd3](https://i.imgur.com/RLL7eEI.jpg)

最后，建议在`/etc/issue`中加入配置的密码，方便后续的人查看默认用户密码。根据参看链接[2]还可以修改`/etc/passwd`的第一行`root:x:...`为`root::...`达到使用root用户的VNC免密登录，但是如果是ssh登录的话，需要在`/etc/ssh/sshd_config`中将`PermitEmptyPasswords no`设置为`PermitEmptyPasswords yes`。

### 2.2.2 通过openstack用户数据修改密码

如果不想修改镜像就可以使用openstack启动实例时导入用户数据的方式来修改密码，加入修改脚本如：

```shell
#!/bin/sh
passwd ubuntu<<EOF
ubuntu
ubuntu
EOF
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service ssh restart
```

如下图所示加入上述脚本，不同的openstack版本此处有所不同，依具体版本操作。

![openstack userdata](https://i.imgur.com/wQ0HrVd.jpg)

此外还有修改openstack的nova.conf和dashboard配置的方式来加入修改密码选项[3]，由于openstack的版本修改该种方式不一定可行，视具体版本处理。

## 2.3 centos镜像

centos官方的镜像地址为：http://cloud.centos.org/centos/7/images/

centos的镜像默认用户是"centos"，处理方式和ubuntu一样，可以通过guestfish或者在创建实例时导入脚本。

同样是使用guetsfish打开镜像然后修改`/etc/cloud/cloud.cfg`文件，如下图所示：

![](https://i.imgur.com/mdfElHS.jpg)

![](https://i.imgur.com/Hmlwt2w.jpg)



【参考链接】

1）[guestfish工具修改ubuntu镜像密码](http://blog.csdn.net/shuaijiasanshao/article/details/51260673)

2）[密码修改](https://ask.openstack.org/en/question/5531/defining-default-user-password-for-ubuntu-cloud-image/)

3）[openstack镜像密码修改](https://xiexianbin.cn/openstack/2017/03/23/OpenStack-image-password-modification)

[返回文首](#jump0)










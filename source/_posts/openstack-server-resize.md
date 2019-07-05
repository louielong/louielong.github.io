---
title: Openstack虚拟机Resize
date: 2019-07-05 11:17:48
tags:
  - Openstack
keywords:
  - vm resize
categories:
  - Openstack
description:
  - Openstack 虚拟机 Resize
summary_img:
---

## 一、前言

最近研究区块链时，突然发现没法连接区块链服务器，一检查发现是区块链程序进程退出了，重启之后仍然退出，再次检查发现是磁盘空间不够了，因为部署在Openstack环境中，因此可以直接使用Openstack的Resize扩展一下磁盘空间，但是点击Resize后并没有改变flavor大小，dashboard上并没有报错，控制节点也看不到错误，进入到计算节点查看才发现有如下报错:

>2019-07-05 11:12:12.599 21343 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/contextlib.py", line 35, in __exit__
>2019-07-05 11:12:12.599 21343 ERROR oslo_messaging.rpc.server     self.gen.throw(type, value, traceback)
>2019-07-05 11:12:12.599 21343 ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/dist-packages/nova/compute/manager.py", line 7958, in _error_out_instance_on_exception
>2019-07-05 11:12:12.599 21343 ERROR oslo_messaging.rpc.server     raise error.inner_exception
>2019-07-05 11:12:12.599 21343 ERROR oslo_messaging.rpc.server ResizeError: Resize error: not able to execute ssh command: Unexpected error while running command.
>2019-07-05 11:12:12.599 21343 ERROR oslo_messaging.rpc.server Command: ssh -o BatchMode=yes 10.0.0.62 mkdir -p /var/lib/nova/instances/6119b573-1b02-4c72-87c9-14e1fd645449
>2019-07-05 11:12:12.599 21343 ERROR oslo_messaging.rpc.server Exit code: 255

错误显示是计算节点*cmp002*无法登录*cpm001*，因此无法在另外一个节点重建虚机。

## 二、处理办法

通过允许计算节点间nova用户免密登录解决。

### 2.1 计算节点间nova用户免密登录

对nova用户进行配置[1]。

### 2.1.1 取消禁止登录权限

使用***root***用户在**所有计算节点**上执行以下操作

```shell
# usermod -s /bin/bash nova
```

检查修改，确保nova用户具备shell执行权限：

```shell
# cat /etc/passwd | grep nova
nova:x:64060:64060::/var/lib/nova:/bin/bash
```

### 2.1.2 配置SSH

在**任一个计算节点**上，用nova用户登录，创建密钥

```shell
# su - nova
$ ssh-keygen -t rsa
```

在该节点上，配置nova用户的SSH配置，不执行主机密钥验证，

```shell
$ cat << EOF > ~/.ssh/config
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
EOF
```

拷贝id_rsa.pub为authorized_keys，并修改权限

```shell
$ cat ~/.ssh/id_rsa.pub > .ssh/authorized_keys
$ chmod 600 .ssh/authorized_keys
```

### 2.1.3 将公钥拷贝至其他所有计算节点

1）直接拷贝公钥

将计算节点的nova根目录下`/var/lib/nova/.ssh/id_rsa.pub`内容拷贝其他所有计算节点的`/var/lib/nova/.ssh/authorized_keys`即可。

2）使用命令上传

*若计算节点的nova用户设置的有密码也可直接用命令上传*，

```shell
nova@cmp002:~$ ssh-copy-id -i ~/.ssh/id_rsa.pub nova@cmp001
```

在任意节点验证，到其他所有计算节点可否SSH登录

```shell
# su - nova
$ ssh nova@cmp001
```

重试resize，即可完成，发现resize完成后，虚拟机迁移到目标计算节点上。

### **【备注】**

以上的操作需要配置计算点间无密码访问，但是那么多计算节点相互访问配置较为麻烦，有提到通过修改nova配置文件，允许在同一台服务器上进行resize和migrate操作配置项，如下[2]：

```shell
# vim /etc/nova/nova.conf

allow_resize_to_same_host=true
allow_migrate_to_same_host=true
```

然后重启nova-api和nova-compute服务：

```shell
# systemctl restart nova-compute
```

```shell
# systemctl restart nova-api
```

但在计算节点配置后，重启nova-compute服务，重试resize没有效果。**后来才发现，这仅适用于计算节点只有一台的测试环境，多计算节点的生产环境并不适用**，仍需要ssh到其他计算节点。





## 【参考文献】

1）[nova对instance做resize操作失败](https://blog.csdn.net/Tech_Salon/article/details/75124135)

2） [Openstack虚拟机Resize失败问题解决方法](http://zjzone.cc/index.php/2017/05/06/openstack-xu-ni-ji-resize-shi-bai-wen-ti-jie-jue-fang-fa/)
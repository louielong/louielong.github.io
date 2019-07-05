---
title: Openstack虚拟机磁盘创建超时
date: 2019-05-09 15:39:03
tags:
  - Openstack
  - cinder
keywords:
  - cinder
categories:
  - Openstack
description:
  - VM磁盘连接超时
summary_img:
---

## 一、前言

最近在创建虚拟机时VM创建非常的慢并且出现创建失败的情况，计算节点报错显示块设备映射超时，在尝试了61次207秒后失败，此时查看Openstack的卷设备创建，发现volume显示是*downloading*状态，即未创建完成。

>ERROR nova.compute.manager [req-94c955d9-9c94-4536-90e2-d21928344444 381fdf4b65aa45ef98a2ad20bc4bd079 4353036cc27542cf84e1bccf1b4bfe33 - default default] [    instance: 655eccd6-cafb-4291-b55a-c53dd000a8e4] Build of instance 655eccd6-cafb-4291-b55a-c53dd000a8e4 aborted: Volume 0e4150db-567f-4ae0-a947-8fc7a0d624f0 did not finish being created    even after we waited 207 seconds or 61 attempts. And its status is downloading.: BuildAbortException: Build of instance 655eccd6-cafb-4291-b55a-c53dd000a8e4 aborted: Volume 0e4150db-567    f-4ae0-a947-8fc7a0d624f0 did not finish being created even after we waited 207 seconds or 61 attempts. And its status is downloading.

该错误原因是Openstack卷创建时间过长与所需卷存储设置大小有关。

## 二、解决办法

### 方案一：增大尝试次数

根据[1]增大`block_device_allocate_retries`参数即可，修改**计算节点**的`/etc/nova/nova.conf`

```yaml
block_device_allocate_retries = 180
```

重启nova服务即可。

### 方案二：使用*Image-Volume cache*

在修改`block_device_allocate_retries`时意外发现注释中有一段标注

>Number of times to retry block device allocation on failures. Starting with
>Liberty, Cinder can use image volume cache. This may help with block device
>allocation performance. Look at the cinder image_volume_cache_enabled
>configuration option.

从L版本Openstack加入了镜像券存储缓存的功能，通过这个功能可以快速创建镜像卷存储。根据文档[2][3]修改**块存储节点**的`/etc/cinder/cinder.conf`，加入以下内容

```yam
[DEFAULT]
...
cinder_internal_tenant_project_id = 4353036cc27542cf84e1bccf1b4bfe33
cinder_internal_tenant_user_id = 381fdf4b65aa45ef98a2ad20bc4bd079

[lvm]
...
image_volume_cache_max_size_gb = 200
image_volume_cache_max_count = 50
image_volume_cache_enabled = True
```

**其中**：

*cinder_internal_tenant_project_id*和*cinder_internal_tenant_user_id*是为了配置块存储的内部租户。这个租户拥有这些缓存并且可以进行管理。这可以保护用户不必看到这些缓存，但是也没有全局隐藏。`cinder_internal_tenant_user_id`选择cinder用户可以保证admin用户看不到缓存镜像，这里设置为admin用户是便于查看修改效果

```shell
xdnsadmin@ctl01:~$ openstack project list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 4353036cc27542cf84e1bccf1b4bfe33 | admin    |

xdnsadmin@ctl01:~$ openstack user list
+----------------------------------+----------+
| ID                               | Name     |
+----------------------------------+----------+
| 381fdf4b65aa45ef98a2ad20bc4bd079 | admin    |
| 6a4d2c1e59364bacadb8e72120b49601 | cinder   |
```

如图所示，`image-88e73002-a4e6-4aff-88cf-2711aa7f24c8`为ubuntu-16.04镜像的缓存文件，当需要创建基于该镜像的新volume时可以快速clone，创建速度大大提升。

![image-volume cache](https://raw.githubusercontent.com/louielong/blogPic/master/20190509160626.png)





【参考链接】

1）[配置block_device_allocate_retries解决卷创建超时的问题](https://www.topomel.com/archives/720.html)

2）[Image-Volume cache](https://docs.openstack.org/cinder/latest/admin/blockstorage-image-volume-cache.html)

3）[image-volume cache翻译](https://www.jianshu.com/p/759cb7efb844)

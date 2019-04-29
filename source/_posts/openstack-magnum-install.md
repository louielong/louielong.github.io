---
title: OpenStack Magnum安装(Rocky)
date: 2019-04-12 17:44:10
tags:
  - Openstack
keywords:
  - Opensatck
  - Magnum
categories:
  - Openstack
description:
  - Openstack magnum 安装 -- 踩坑与填坑系列
summary_img:
  - https://www.openstack.org/software/images/mascots/magnum.png
---

## 前言

最近在研究OpenStack和Kubernetes融合相关的方法，通过查阅[1]发现openstack与k8s融合无外乎两种方法：一是K8S部署在OpenStack平台之上，二是K8S和OpenStack组件集成。方案一的优点是k8s容器借用虚机的多租户具有很好的隔离性，但缺点也很明显K8S使用的是在虚机网络之上又嵌套的一层overlay网络，k8s的网络将难以精细化的管控。方案二将k8s与openstack的部分组件进行集成，是的K8S与openstack具备很好的融合性，从而实现1+1>2的效果。openstack中的magnum是一个典型的解决方案，通过magnum可以镜像容器编排，同时可以快速部署一个Swarm、K8S、Mesos集群。

本文仅介绍如何在openstack中安装部署magnum以及安装中遇到的问题。

此外，**我还想吐槽一句**，`OpenStack`社区从`Pike`版本开始弃用了`auth_uri`以及keystone的`35357`端口全部改为公开的`5000`端口和使用`www_authenticate_uri`来进行keystone认证，社区的核心组件已经更换完成，但是非核心组件并未完全跟进，如`Magnum-api`、`Heat`安装时就出现过这类问题。

## 一、Magnum安装准备

根据官方组件说明[2]，必须的组件有：

- Keystone-Identity service
- Glance-Image service
- Nova-Compute service
- Neutron-Networking service
- Heat-Orchestration service。

可选组件有：

- Cinder-Block Storage service
- Octavia-[Load Balancer as a Service (LBaaS v2)](http://docs.openstack.org/networking-guide/config-lbaas.html)
- Barbican-[Key Manager service](http://docs.openstack.org/project-install-guide/key-manager/draft/)
以及
- Swift-[Object Storage service](http://docs.openstack.org/project-install-guide/object-storage/draft/)
- Ironic-[Bare Metal service](http://docs.openstack.org/project-install-guide/baremetal/draft/)

![Magnum Depends on](https://raw.githubusercontent.com/louielong/blogPic/master/imgMagnum_depends.png)

**本次安装使用的Rocky版本，Neutron网络架构采用的是ovs+vxlan的方式**

## 二、安装Magnum

我的openstack环境采用纯手工的方式按照官方指导文档进行安装，同样Magnum也是Manual的方式安装，系统版本为`Ubuntu-18.04 LTS`，因为Rocky版本的`deb`包只有18.04才有。Magnum-api 版本为`7.0.1-0ubuntu1~cloud0`。

按照指导手册[3]，进行安装，安装和配置方式极其简单，**但是**官方的配置文档没有来的及更新，以及deb包更新不及时遇到了很多奇奇怪怪的bug，在查找原因时也很难确定关键词而无法搜索到正确的答案。

按照官方文档在安装完Magnum后通过`openstack coe service list`来查看是否安装成功，如果该命令执行失败通过查看`/var/log/magnum/magnum-api.log`来排查问题。**但是**，该命令执行成功也不代表这Magnum的服务能够正常使用，后续使用中主要查看的是Magnum调度的日志`/var/log/magnum/magnum-conductor.log`。

### 2.1  magnum-api 报错

Magnum API报错 ，如下所示，根据[4]，新的稳定版代码已修复但是安装的deb包并不是最新的代码

>  File "/usr/lib/python2.7/dist-packages/magnum/common/keystone.py", line 47, in auth_url
>       return CONF[ksconf.CFG_LEGACY_GROUP].auth_uri.replace('v2.0', 'v3')
>
>  AttributeError: 'NoneType' object has no attribute 'replace'

解决方法如下如[Add support for www_authentication_uri](https://review.openstack.org/#/c/614082/)所示，添加如下patch

```shell
vim /usr/lib/python2.7/dist-packages/magnum/common/keystone.py +47

#return CONF[ksconf.CFG_LEGACY_GROUP].auth_uri.replace('v2.0', 'v3')
conf = CONF[ksconf.CFG_LEGACY_GROUP]
auth_uri = (getattr(conf, 'www_authenticate_uri', None) or
            getattr(conf, 'auth_uri', None))
if auth_uri:
   auth_uri = auth_uri.replace('v2.0', 'v3')
return auth_uri
```

修改完成后需要重启magnum服务，否则修改的代码不生效。

```shell
service magnum-* restart
```

### 2.2 指定swarm集群的volumed size

按照文档创建一个集群模板时出现如下报错：

```shell
root@ctl01:/home/xdnsadmin# openstack coe cluster template create swarm-cluster-template --image fedora-atomic-latest --external-network Provider --dns-nameserver 223.5.5.5 --master-flavorm1.small --flavor m1.small --coe swarm
docker volume size None GB is not valid, expecting minimum value 3GB for devicemapper storage driver (HTTP 400) (Request-ID: req-392aaa81-d44c-48cc-8800-2e9697539a28)
```

解决办法是指定一个volume size，按照官方的说法没有cinder服务也可以使用magnum，**这个我并未验证**，我的集群安装的cinder服务。

```shell
openstack coe cluster template create swarm-cluster-template --image fedora-atomic-latest --external-network Provider --dns-nameserver 223.5.5.5 --master-flavor m1.small --flavor m1.small --coe swarm --docker-volume-size 10
```

同时这里仍有一个问题，查看*2.5*

### 2.2 docker volume_type

在安装过程出现无效的volumetype的错误，按照[5]需要设置volumetype同时进行相关配置

>ERROR heat.engine.resource     raise exception.ResourceFailure(message, self, action=self.action)
>ERROR heat.engine.resource ResourceFailure: resources[0]: Property error: resources.docker_volume.properties.volume_type: Error validating value '': The VolumeType () could not be found.

1)Check if you have any volume types defined.

```shell
openstack volume type list
```

2)If there are none, create one:

```shell
openstack volume type create volType1 --description "Fix for Magnum" --public
```

3)Then in  /etc/magnum/magnum.conf add this line in the [cinder] section:

```shell
default_docker_volume_type = volType1
```

### 2.3  `Barbican`服务未安装

按照文档在选择`[certificates]`时使用推荐的`barbican`来存储认证证书，而我的openstack集群并未安装`barbican`服务，因此收到了如下报错，但是我当时并未发现原因所在，我一直以为是keystone的`publicURL endpoint `无法访问。

>2019-04-09 18:42:26.505 32428 ERROR magnum.conductor.handlers.common.cert_manager [req-ffd4b077-80ee-4846-bdc0-84817aa699a5 - - - - -] Failed to generate certificates for Cluster: 03cd9a72-c980-4947-b273-d9bcb88a6913: AuthorizationFailure: unexpected keystone client error occurred: publicURL endpoint for key-manager service not found
>..........
>2019-04-09 18:42:26.505 32428 ERROR magnum.conductor.handlers.common.cert_manager     connection = get_admin_clients().barbican()
>2019-04-09 18:42:26.505 32428 ERROR magnum.conductor.handlers.common.cert_manager   File "/usr/lib/python2.7/dist-packages/magnum/common/exception.py", line 65, in wrapped
>2019-04-09 18:42:26.505 32428 ERROR magnum.conductor.handlers.common.cert_manager     % sys.exc_info()[1])
>2019-04-09 18:42:26.505 32428 ERROR magnum.conductor.handlers.common.cert_manager AuthorizationFailure: unexpected keystone client error occurred: publicURL endpoint for key-manager service not found

该问题的解决方法很简单，安装`barbican` 服务，或者使用`x509keypair`或`local`方式存储证书

```shell
[certificates]
cert_manager_type = barbican # x509keypair or local
storage_path = /var/lib/magnum/certificates/
```

若使用local方式，还需创建目录`/var/lib/magnum/certificates/`，并将其所属组改为magnum，`sudo chown -R magnum:magnum /var/lib/magnum/certificates/`，否则会出现如下报错：

> 2019-04-10 15:40:50.675 20235 ERROR magnum.conductor.handlers.common.cert_manager [req-c31e7588-a486-4ea8-8adc-16ff8f37b284 - - - - -] Failed to generate certificates for Cluster: e579     0f96-e8c5-49c5-82f9-87faf8438585: CertificateStorageException: Could not store certificate: [Errno 2] No such file or directory: '/var/lib/magnum/certificates/5e8beb02-9378-4e8c-8b4d-1     cae933de665.crt'

同时，在修复此问题的过程中查看到一个警告，该警告对于使用过程也造成了很大的影响`Auth plugin and its options for service user must be provided in [keystone_auth] section. Using values from [keystone_authtoken] section is deprecated.: MissingRequiredOptions: Auth plugin requires parameters which were not given: auth_url`

需要在`/etc/magnum/magnum.conf `中 添加`keystone_auth`项 ，此外，这里的还涉及到接下来的问题*2.4*

```shell
[keystone_auth]
auth_version = v3
auth_url = http://10.253.0.61:5000/v3
#www_authenticate_uri = http://ctl01:5000/v3
auth_uri = http://10.253.0.61:5000
project_domain_id = default
project_name = service
user_domain_id = default
password = magnum
username = magnum
auth_type = password
```

### 2.4 swarm集群创建超时

在处理完以上的magnum conductor问题后，开始创建swarm集群，但是出现创建超时(超时时长为60分钟)集群的stack启动失败的问题。原因是按照之前的官方文档创建所有服务的endpoint都是使用`http://ctl01:xxx`而此处创建的swarm集群需要与opensatck相关服务进行通信，无法识别`ctl01`域名，解析不到服务地址IP，因此**需要将所有的endpiont服务的public地址都是用公开的IP替换**。

>ERROR oslo_messaging.rpc.server [req-4a8e8e2d-cae0-4113-a49d-57bb91c03b8d - - - - -] Exception during message handling: EndpointNotFound: http://ctl01:9511/v1 endpoint for identity service not found
>....
>ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/dist-packages/magnum/conductor/handlers/cluster_conductor.py", line 68, in cluster_create
>ERROR oslo_messaging.rpc.server     cluster_driver.create_cluster(context, cluster, create_timeout)
>....
>ERROR oslo_messaging.rpc.server   File "/usr/lib/python2.7/dist-packages/keystoneauth1/access/service_catalog.py", line 464, in endpoint_data_for
>ERROR oslo_messaging.rpc.server     raise exceptions.EndpointNotFound(msg)
>ERROR oslo_messaging.rpc.server EndpointNotFound: http://ctl01:9511/v1 endpoint for identity service not found

使用命令`openstack endpoint list | grep public`查看相关的endpoint。

### 2.5 swarm-cluster启动失败

在解决完*2.4*的问题后又迎来了新的问题，swarm-cluster集群启动创建成功，但是服务报错。这是能够通过`floating ip`登录swarm master节点使用`fedora`用户以及密钥`mykey`登录。查看`cloud init`日志`/var/log/cloud-init-output.log`，会发现如下的报错：

>/var/lib/cloud/instance/scripts/part-006: line 13: /etc/etcd/etcd.conf: No such file or directory
>/var/lib/cloud/instance/scripts/part-006: line 26: /etc/etcd/etcd.conf: No such file or directory
>/var/lib/cloud/instance/scripts/part-006: line 38: /etc/etcd/etcd.conf: No such file or directory

`etcd`初始失败从而导致`swarm-manager`启动失败，这个问题还是相当的严重的，因为很难排查原因，最终在opensatck的讨论邮件列表中找到的答案[6]，根本原因是`coe`使用错误，应该使用`swarm-mode`而不是安装手册里写的`swam`，这点实在是**太坑人**了。

> $openstack coe cluster template show docker-swarm
>
> | docker_storage_driver | devicemapper                         |
> | network_driver             | docker                                     |
> | coe                               | swarm-mode                           |
>
>Never got the "swarm" driver to work, you should use "swarm-mode" instead which uses native Docker clustering without etcd.

**同时需要注意的是**，使用的`Fedora`镜像必须是`Fedora-Atomic-27-xx`，`Frdora 27`之后的镜像改变较大，`cloud init`脚本无法正常工作。还需添加必要的patch [swarm-mode allow TCP port 2377 to swarm master node](https://review.openstack.org/#/c/598179/)，修改`magnum/drivers/swarm_fedora_atomic_v2/templates/swarmcluster.yaml`的Line 247，添加一下内容。

```yaml
        - protocol: tcp
          port_range_min: 2377
          port_range_max: 2377
```

随后，

1）重新创建`swarm-cluster`模板


```shell
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# openstack coe cluster template create swarm-cluster-template --image fedora-atomic-latest --external-network Provider --dns-nameserver 223.5.5.5 --master-flavor m1.small --flavor m1.small --coe swarm-mode  --docker-volume-size 10
```

![创建swarm-cluster集群模板](https://raw.githubusercontent.com/louielong/blogPic/master/imgcreate_swarm_cluster_template.png)

2）启动`swarm-cluster`实例

```shell
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# openstack coe cluster create swarm-cluster --cluster-template swarm-cluster-template --master-count 1 --node-count 1 --keypair mykey
```

等待集群实例创建完成

![show swarm_cluster](https://raw.githubusercontent.com/louielong/blogPic/master/imgshow_swarm_cluster.png)

3）配置`swarm-cluster`访问

首先，生成证书，然后导入环境变量即可访问swarm集群(*控制节点需要安装docker客户端*)

```shell
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# $(openstack coe cluster config swarm-cluster --dir /home/xdnsadmin/workplace/magnum/swarm/cluster)
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# export DOCKER_HOST=tcp://10.253.0.115:2375
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# export DOCKER_CERT_PATH=/home/xdnsadmin/workplace/magnum/swarm/cluster
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# export DOCKER_TLS_VERIFY=True
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# docker info
Containers: 0
.....
Images: 0
Server Version: 1.13.1
Storage Driver: devicemapper
 Pool Name: docker-docker--pool
.....
 Backing Filesystem: xfs
 Data file:
 Metadata file:
 ......
 Library Version: 1.02.144 (2017-10-06)
Logging Driver: journald
Cgroup Driver: systemd
Plugins:
 Volume: local
 Network: bridge host macvlan null overlay
 Authorization: rhel-push-plugin
 Log:
Swarm: active
 NodeID: jcsq6ms8717ougczpyhgdayic
 Is Manager: true
 ClusterID: jxzj1kr158cuqu5ph8k2nq8pu
 Managers: 1
 Nodes: 2
 Orchestration:
  Task History Retention Limit: 5
 Raft:
  Snapshot Interval: 10000
  Number of Old Snapshots to Retain: 0
  Heartbeat Tick: 1
  Election Tick: 3
 Dispatcher:
  Heartbeat Period: 5 seconds
 CA Configuration:
  Expiry Duration: 3 months
  Force Rotate: 0
 Autolock Managers: false
 Root Rotation In Progress: false
 Node Address: 10.0.0.8
 Manager Addresses:
  10.0.0.8:2377
Runtimes: oci runc
Default Runtime: oci
.........
  Profile: /etc/docker/seccomp.json
 selinux
Kernel Version: 4.14.18-300.fc27.x86_64
Operating System: Fedora 27 (Atomic Host)
OSType: linux
Architecture: x86_64
CPUs: 1
Total Memory: 1.946GiB
Name: swarm-cluster-cgjrw2xrwylk-primary-master-0.novalocal
ID: 27CS:YZSQ:WOQJ:PMHZ:ALTJ:WMB3:IIA7:FLUC:2RIW:4CKA:FZWV:4GZA
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false

root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# docker -H 10.253.0.115:2375 run busybox:latest echo 'Hello world!'
Unable to find image 'busybox:latest' locally
Trying to pull repository docker.io/library/busybox ...
Trying to pull repository registry.fedoraproject.org/busybox ...
Trying to pull repository registry.access.redhat.com/busybox ...
Trying to pull repository docker.io/library/busybox ...
sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd: Pulling from docker.io/library/busybox
fc1a6b909f82: Pull complete
Digest: sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd
Status: Downloaded newer image for docker.io/busybox:latest
Hello world!

```

### 2.6 standard_init_linux.go:178: exec user process caused "permission denied"

该错误与swarm节点的存储格式有关，由于之前解决swarm无法启动而被带偏添加了`--docker-storage-driver overlay`参数在`swarm-cluster`模板中，导致容器无法启动，通过查看[7]得知与其支持的存储驱动有关

```shell
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# docker -H 10.253.0.111:2375 run busybox echo "Hello from Docker!"
Unable to find image 'busybox:latest' locally
Trying to pull repository docker.io/library/busybox ...
Trying to pull repository registry.fedoraproject.org/busybox ...
Trying to pull repository registry.access.redhat.com/busybox ...
Trying to pull repository docker.io/library/busybox ...
sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd: Pulling from docker.io/library/busybox
fc1a6b909f82: Pull complete
Digest: sha256:954e1f01e80ce09d0887ff6ea10b13a812cb01932a0781d6b0cc23f743a874fd
Status: Downloaded newer image for docker.io/busybox:latest
standard_init_linux.go:178: exec user process caused "permission denied"
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# docker -H 10.253.0.111:2375 ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS               NAMES
f1a60f01625f        busybox             "echo 'Hello from Do…"   10 minutes ago      Exited (1) 10 minutes ago                       elastic_kirch
root@ctl01:/home/xdnsadmin/workplace/magnum/swarm/cluster# docker -H 10.253.0.111:2375 logs f1a60f01625f
standard_init_linux.go:178: exec user process caused "permission denied"
```

目前`Docker`支持如下几种storage driver：

| Technology    | Storage driver name |
| :------------ | :------------------ |
| OverlayFS     | overlay             |
| AUFS          | aufs                |
| Btrfs         | btrfs               |
| Device Mapper | devicemapper        |
| VFS           | vfs                 |
| ZFS           | zfs                 |

详细的对比可以参看：[Docker之几种storage-driver比较](https://blog.csdn.net/vchy_zhao/article/details/70238690)


以上，就是Openstack Rocky版本Magnum安装与排坑的过程，不说啦，刚刚安装完Rocky版本，Stein版本就放出来了，我接着去踩雷啦 - -！**开源真香**

![无限 Fix Bug](https://raw.githubusercontent.com/louielong/blogPic/master/imgFix_bug.gif)



【参考链接】

1）[如何把OpenStack和Kubernetes结合在一起来构建容器云平台](https://www.zhihu.com/question/63576896)

2）[Magnum -Container Orchestration Engine Provisioning](https://www.openstack.org/software/releases/rocky/components/magnum)

3）[Magnum安装手册](https://docs.openstack.org/magnum/rocky/install/install.html)

4）[magnum-api not working with www_authenticate_uri](https://bugs.launchpad.net/ubuntu/+source/magnum/+bug/1793813)

5）[magnum cluster create k8s cluster Error: ResourceFailure](https://ask.openstack.org/en/question/110729/magnum-cluster-create-k8s-cluster-error-resourcefailure/)

6）[Fwd: openstack queens magnum error](http://lists.openstack.org/pipermail/openstack-discuss/2018-December/000937.html)

7）[standard_init_linux.go:175: exec user process caused "permission denied"](https://github.com/moby/moby/issues/26495)

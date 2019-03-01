---
title: k8s集群安装与部署
date: 2019-03-01 09:38:08
tags:
  - k8s
keywords:
  - k8s
categories:
  - k8s
description:
  - k8s集群部署与安装
summary_img:
  - https://i.imgur.com/pQ6BiOT.png
---



## 一、前言

一年前研究过一段时间的K8S，最开始看k8s的概念一头雾水，随着学习的深入有一些了解，跟着[cloudman](https://steemit.com/@cloudman6)的教程尝试搭建k8s集群，但是当时由于墙的原因搭建一个k8s集群颇为费劲，在做完实验对k8s有一个了解后就没有继续深入。最近由于新的项目需要重新研究k8s环境，正好做一下笔记，记录搭建的过程，在搭建过程中一个好消息就是k8s的docker容器在国内有镜像提供商，像阿里云、dockerhub上都有，简直是一大**救星**。**由于有一定的知识了解，本文不再介绍k8s的基本概念和相关的命令含义。**

搭建过程主要参考的是molscar的[使用kubeadm 部署 Kubernetes(国内环境)](https://juejin.im/post/5b8a4536e51d4538c545645c)。

## 二、安装环境准备

k8s集群的安装，至少需要一个master节点和一个slave节点，节点配置如下

- 操作系统要求
  - Ubuntu 16.04+
  - Debian 9
  - CentOS 7
  - RHEL 7
  - Fedora 25/26 (best-effort)
  - HypriotOS v1.0.1+
  - Container Linux (tested with 1800.6.0)
- 2+ GB RAM
- 2+ CPUs
- 特定的端口开放（安全组和防火墙未将其排除在外）
- 关闭Swap交换分区

本文搭建时使用的系统是ubuntu 16.04.5，一个master节点4个slave节点，安装的是当前k8s最新的v1.13版本。

### 2.1 docker 安装

#### 1）docker软件安装

**master和slave节点都需要安装docker**

我的网络在教育网访问更快，这里使用[清华源](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)来进行安装

安装依赖：

```shell
sudo apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common
```

信任 Docker 的 GPG 公钥:

```shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

对于 amd64 架构的计算机，添加软件仓库:

```bash
sudo add-apt-repository \
   "deb [arch=amd64] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

最后安装

```shell
sudo apt-get update
sudo apt-get install docker-ce
```

安装完成后通过命令查看

```shell
xdnsadmin@k8smaster:~$ docker --version
Docker version 18.09.2, build 6247962
```

#### 2）免sudo执行docker命令

为了使普通用户免sudo使用docker命令

将当前用户加入 `docker` 组：

```shell
sudo usermod -aG docker $USER
```

#### 3）镜像加速

对于使用 [systemd](https://link.juejin.im?target=https%3A%2F%2Fwww.freedesktop.org%2Fwiki%2FSoftware%2Fsystemd%2F) 的系统，请在 `/etc/docker/daemon.json` 中写入如下内容（如果文件不存在请新建该文件）

这里使用Docker 官方加速器 ，也可以使用阿里云或者中科大的加速

```yaml
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```

重新启动服务

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### 4）禁用 swap

**该步骤只需要在master节点执行即可**

> 对于禁用`swap`内存，具体原因可以查看Github上的Issue：[Kubelet/Kubernetes should work with Swap Enabled](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fkubernetes%2Fissues%2F53533)

临时关闭方式：

```shell
sudo swapoff -a
```

永久关闭方式：

编辑`/etc/fstab`文件，注释掉引用`swap`的行

```shell
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/ubuntu--vg-root /               ext4    errors=remount-ro 0       1
# /boot was on /dev/sda2 during installation
UUID=50bd7683-2bf2-4953-bf0b-125f74e06e52 /boot           ext2    defaults        0       2
# /boot/efi was on /dev/sda1 during installation
UUID=6EA7-687A  /boot/efi       vfat    umask=0077      0       1
#/dev/mapper/ubuntu--vg-swap_1 none            swap    sw              0       0
```

随后重启机器

测试：输入`top` 命令，若 KiB Swap一行中 total 显示 0 则关闭成功

### 2.2 安装 kubeadm, kubelet 以及 kubectl

**master和slave节点都需要安装三个软件**

#### 1）k8s加速安装配置

这里可以选择使用清华源或中科大源加速安装

首先添加软件源认证

```shell
apt-get update && apt-get install -y apt-transport-https curl
curl -s http://packages.faasx.com/google/apt/doc/apt-key.gpg | sudo apt-key add -
```

随后添加中科大源仓库地址并安装软件

```
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

### 2.3 k8s集群安装

#### 1）cgroups配置

在Master节点中配置 cgroup driver

查看 Docker 使用 cgroup driver:

```shell
docker info | grep -i cgroup
-> Cgroup Driver: cgroupfs
```

而 kubelet 使用的 cgroupfs 为system，不一致故有如下修正：

```shell
sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

加上如下配置：

```shell
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
```

或者

```shell
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
```

重启 kubelet

```shell
systemctl daemon-reload
systemctl restart kubelet
```

#### 2）镜像配置

这里使用`kubeadm`来进行自动化集群安装和配置，在安装过程中会从google仓库中下载镜像，由于墙的原因镜像下载会失败，这里预选将相应的镜像下载下来再重新改tag

Master 和 Slave 启动的核心服务分别如下：

| Master 节点                      | Slave 节点                       |
| -------------------------------- | -------------------------------- |
| etcd-master                      | Control plane(如：calico,fannel) |
| kube-apiserver                   | kube-proxy                       |
| kube-controller-manager          | other apps                       |
| kube-dns                         |                                  |
| Control plane(如：calico,fannel) |                                  |
| kube-proxy                       |                                  |
| kube-scheduler                   |                                  |

使用如下命令：

```shell
kubeadm config images list
```

获取当前版本kubeadm 启动需要的镜像，示例如下：

```shell
xdnsadmin@k8smaster:~$ kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.13.3
k8s.gcr.io/kube-controller-manager:v1.13.3
k8s.gcr.io/kube-scheduler:v1.13.3
k8s.gcr.io/kube-proxy:v1.13.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.2.24
k8s.gcr.io/coredns:1.2.6

```

从`dockerhub`的[mirrorgooglecontainers](https://hub.docker.com/u/mirrorgooglecontainers)仓库下载相应镜像，如

```shell
xdnsadmin@k8smaster:~$ docker pull mirrorgooglecontainers/kube-apiserver:v1.13.3
....
xdnsadmin@k8smaster:~$ docker tag mirrorgooglecontainers/kube-apiserver:v1.13.3 k8s.gcr.io/kube-apiserver:v1.13.3
xdnsadmin@k8smaster:~$ docker rmi mirrorgooglecontainers/kube-apiserver:v1.13.3
```

这里分享一个脚本来自动完成镜像下载并改tag的操作

```shell
xdnsadmin@k8smaster:~$ cat k8s_images_get.sh
#!/bin/bash -e
#########################################################################
# File Name: k8s_images_get.sh
# Author: louie.long
# Mail: ylong@biigroup.cn
# Created Time: Mon 25 Feb 2019 05:05:54 PM CST
# Description:
#########################################################################

if [ ! -x /usr/bin/kubeadm ]; then
	echo "***please install kubeadm first***"
fi

while read IMAGE; do
	IMG=`echo $IMAGE | sed "s/k8s\.gcr\.io/mirrorgooglecontainers/"`
	echo $IMG
done < <(kubeadm config images list)
```

#### 3）检查端口占用

Master 节点

| Protocol | Direction | Port Range | Purpose                 | Used By              |
| -------- | --------- | ---------- | ----------------------- | -------------------- |
| TCP      | Inbound   | 6443*      | Kubernetes API server   | All                  |
| TCP      | Inbound   | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
| TCP      | Inbound   | 10250      | Kubelet API             | Self, Control plane  |
| TCP      | Inbound   | 10251      | kube-scheduler          | Self                 |
| TCP      | Inbound   | 10252      | kube-controller-manager | Self                 |

Worker节点

| Protocol | Direction | Port Range  | Purpose             | Used By             |
| -------- | --------- | ----------- | ------------------- | ------------------- |
| TCP      | Inbound   | 10250       | Kubelet API         | Self, Control plane |
| TCP      | Inbound   | 30000-32767 | NodePort Services** | All                 |

#### 4）初始化 kubeadm

在初始化过程中需要`coredns`的支持，同样需要预先下载，随着安装的k8s版本不同所需的`coredns`版本也会相应改变，可以先执行`kubeadm init`查看报错

```shell
xdnsadmin@k8smaster:~$ docker pull docker pull coredns/coredns:1.2.6
xdnsadmin@k8smaster:~$ docker tag coredns/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6
xdnsadmin@k8smaster:~$ docker rmi coredns/coredns:1.2.6
```

集群master节点初始化：

```bash
xdnsadmin@k8smaster:~$ sudo kubeadm init --apiserver-advertise-address=<your ip> --pod-network-cidr=10.244.0.0/16
......
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [Podnetwork].yaml" with one of the addon options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

**init 常用主要参数：**

- --kubernetes-version: 指定Kubenetes版本，如果不指定该参数，会从google网站下载最新的版本信息。
- --pod-network-cidr: 指定pod网络的IP地址范围，它的值取决于你在下一步选择的哪个网络网络插件，这里选用flannel网络因此指定`--pod-network-cidr=10.244.0.0/16`，需要注意的该CIDR不可修改。
- --apiserver-advertise-address: 指定master服务发布的Ip地址，如果不指定，则会自动检测网络接口，通常是内网IP。
- --feature-gates=CoreDNS: 是否使用CoreDNS，值为true/false，CoreDNS插件在1.10中提升到了Beta阶段，最终会成为Kubernetes的缺省选项。

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

master节点测试：

```shell
curl https://127.0.0.1:6443 -k 或者 curl https://<master-ip>:6443 -k
```

回应如下：

```shell
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

**重置初始化**

```shell
xdnsadmin@k8smaster:~$ kubeadm reset
```

**忘记了token，可以通过命令查看**

```shell
xdnsadmin@k8smaster:~$ kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
zsrpzu.lh831z4t4ht3vm1k   15h       2019-03-01T09:40:53+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```

#### 5）安装pod插件

插件可选类型有很多，参考[官方pod网络插件](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network)，这里选用`flannel`网络，网络选择在后续可以修改

```shell
xdnsadmin@k8smaster:~$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

#### 6）节点加入

在slave节点执行一下命令即可自动加入集群

```shell
 kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

查看节点加入情况

```shell
xdnsadmin@k8smaster:~$ kubectl get nodes
NAME        STATUS   ROLES    AGE     VERSION
k8slave1    Ready    <none>   8h      v1.13.3
k8slave2    Ready    <none>   7h59m   v1.13.3
k8slave3    Ready    <none>   7h58m   v1.13.3
k8slave4    Ready    <none>   7h58m   v1.13.3
k8smaster   Ready    master   8h      v1.13.3
```

若有节点未加入查看相关节点的镜像是否下载成功。

## 三、dashboard安装

### 3.1 dashboard安装

k8s集群安装后可以安装dashboard面板便于查看

官方教程：[github.com/kubernetes/…](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fdashboard%2Fwiki%2FInstallation)

#### 1）预备镜像

```shell
k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1

# 下面为插件的镜像
k8s.gcr.io/heapster-amd64:v1.5.4
k8s.gcr.io/heapster-grafana-amd64:v5.0.4
k8s.gcr.io/heapster-influxdb-amd64:v1.5.2
```

同样的方式先从`dockerhub`上下载镜像再改tag

安装dashboard，**不同版本配置模板的链接会有所修改，以官方的安装链接为准**

```shell
xdnsadmin@k8smaster:~$ kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

安装过程中可能出现dashboard重新下载而无法连接`gcr.k8s.io`的情况，则将部署配置文件下载下来手动修改镜像名称

```shell
xdnsadmin@k8smaster:~$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
xdnsadmin@k8smaster:~$ vim kubernetes-dashboard.yaml
xdnsadmin@k8smaster:~$ kubectl create -f kubernetes-dashboard.yaml
```

#### 2）配置访问端口

将`type: ClusterIP`改成`type: NodePort`

```shell
xdnsadmin@k8smaster:~$ kubectl -n kube-system edit service kubernetes-dashboard
# 编辑内容如下：
  ports:
  - nodePort: 32576
    port: 443
    protocol: TCP
    targetPort: 8443
  type: NodePort
```

查询dashboard状态

```shell
xdnsadmin@k8smaster:~/workplace/k8s/dashboard$ kubectl -n kube-system get service kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.97.229.208   <none>        443:31090/TCP   7h47m
```

可以看到k8s将集群内部的`10.97.229.208:443`端口映射到`31090`端口，通过浏览器访问`https://<Mater IP>:31090`

![dashboard login](https://i.imgur.com/HmVNgrV.png)

#### 3）配置admin

创建`kubernetes-dashboard-admin.yaml`并填入以下内容

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
```

之后：`kubectl create -f kubernetes-dashboard-admin.yml`

#### 4) 登录方式

**Kubeconfig登录**

创建 admin 用户

file: admin-role.yaml

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

随后`kubectl create -f admin-role.yaml`

```shell
xdnsadmin@k8smaster:~/workplace/k8s/dashboard$ kubectl -n kube-system get secret|grep admin-token
admin-token-rjzkq                                kubernetes.io/service-account-token   3      4h40m
kubernetes-dashboard-admin-token-2k7mr           kubernetes.io/service-account-token   3      4h41m
xdnsadmin@k8smaster:~/workplace/k8s/dashboard$ kubectl -n kube-system describe secret admin-token-rjzkq
Name:         admin-token-rjzkq
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: e109d87b-3b1c-11e9-84b9-dcda8043f2e7

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1yanprcSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImUxMDlkODdiLTNiMWMtMTFlOS04NGI5LWRjZGE4MDQzZjJlNyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbiJ9.vAasQjLsTSXhaPAVYbB-7m5QSyMxpcTvUveSi6KJFLvZPR1HILA3m7riAEqWnlX4dg8d02JSpPD0rPXLY636j7ppMN64RJcqMF_Ik3OhbSQpPwpx3LF2h94EwGsCKpqdbzlgKG31OOwO4OibHGiCn4DwIZGIZAFP4qx_CMtDg1d4XMOQgYGXKXmgPY98KJ5hxhzrC3lwf7qopiQ3RuyYn9fvho2ZLyy2X0_6r1a1ZZGJSscdlykd7q9u-i5MLl6GTxnjnj5SH_KI4tXP8TknXSTegBreL45Sd_pn5dhGmo4msWB_jOBqi6K5pX3vxm9cjNKibSZ1TuVgf3LFy0VmNw
```

设置 Kubeconfig文件

```
copy ~/.kube/config Kubeconfig
vim Kubeconfig
# 将上文中获取的token加入其中
```

内容如下：

![Kubeconfig file](https://i.imgur.com/qp8kCvV.png)



**Token登录**

获取token:

```shell
xdnsadmin@k8smaster:~/workplace/k8s/dashboard$ kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
```

获取admin-token:

```shell
xdnsadmin@k8smaster:~/workplace/k8s/dashboard$ kubectl -n kube-system describe secret/$(kubectl -n kube-system get secret | grep kubernetes-dashboard-admin | awk '{print $1}') | grep token
```

### 3.2 集成 heapster监控

当前`heapster`已经被废弃，但是还有维护，这里也仍然可以使用。

![Kubeconfig file](https://pic3.zhimg.com/80/v2-6a949839753806ca26da14a552f12f76_hd.jpg)

#### 1）安装 heapster

```shell
mkdir heapster
cd heapster
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/grafana.yaml
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
wget https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

之后修改 heapster.yaml

```shell
--source=kubernetes:https://10.0.0.1:6443   --------改成自己的ip
--sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
```

或者

```shell
--source=kubernetes.summary_api:https://kubernetes.default.svc?inClusterConfig=false&kubeletHttps=true&kubeletPort=10250&insecure=true&auth=
--sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086
```

**修改镜像为`mirrorgooglecontainers`**

```shell
sed -i "s/k8s.gcr.io/mirrorgooglecontainers/"  *.yaml
```

**添加heapster api访问权限**，修改heapster-rbac.yaml文件

```shell
xdnsadmin@k8smaster:~/workplace/k8s/heapster$ cat heapster-rbac.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
# 增加以下内容
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster-kubelet-api
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubelet-api-admin
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
```

随后部署应用

```shell
kubectl create -f .
```

效果如下：

![dashboard1](https://i.imgur.com/xwKiU9e.png)

## 四、测试应用部署

选用[Sock Shop](https://link.juejin.im?target=https%3A%2F%2Fmicroservices-demo.github.io%2F)进行应用部署测试，演示一个袜子商城，以微服务的形式部署起来

```shell
kubectl create namespace sock-shop

kubectl apply -n sock-shop -f "https://github.com/microservices-demo/microservices-demo/blob/master/deploy/kubernetes/complete-demo.yaml?raw=true"
```

等待 Pod 变为 Running 状态，便安装成功

```shell
xdnsadmin@k8smaster:~/workplace/k8s/heapster$ kubectl -n sock-shop get deployment
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
carts          1/1     1            1           137m
carts-db       1/1     1            1           137m
catalogue      1/1     1            1           137m
catalogue-db   1/1     1            1           137m
front-end      1/1     1            1           137m
orders         1/1     1            1           137m
orders-db      1/1     1            1           137m
payment        1/1     1            1           137m
queue-master   1/1     1            1           137m
rabbitmq       1/1     1            1           137m
shipping       1/1     1            1           137m
user           1/1     1            1           137m
user-db        1/1     1            1           137m
```

![sockshop1](https://i.imgur.com/bmCoQw9.png)

官方已将其端口做了映射，不需要修改即可直接访问。如果未做端口映射手动修改即可

```shell
xdnsadmin@k8smaster:~$ kubectl -n sock-shop edit svc front-end
.....

  selector:
    name: front-end
  sessionAffinity: None
  # 将ClusterIP 改成 NodePort
  type: NodePort
status:
  loadBalancer: {}
```

如果加购物车功能够跑通，就说明集群搭建成功

![sockshop](https://i.imgur.com/Z1mXBQQ.png)

卸载 socks shop: `kubectl delete namespace sock-shop`







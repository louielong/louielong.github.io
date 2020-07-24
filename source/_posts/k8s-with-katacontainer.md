---
title: K8S搭配Kata container
date: 2020-07-20 16:47:56
tags:
 - k8s
 - docker
 - kata container
keywords:
 - k8s
 - kata container
categories:
 - K8S
description:
 - K8S 搭配 Kata container
summary_img:
 - https://raw.githubusercontent.com/louielong/blogPic/master/imgkata-containers-compared-to-docker-kubernetes.png
---

## 一、前言

工作中有聊到容器安全相关的话题，专门研究了一下容器安全见上一篇博文[Docker容器安全性分析](container-security.html)，文章详尽的分析了容器相关的安全如图所示：

![容器安全](https://raw.githubusercontent.com/louielong/blogPic/master/imgcontainer-security.png)

针对其中的容器逃逸问题，之前关注过Openstack的一个项目[Kata container](https://katacontainers.io/)项目：

![kata container](https://raw.githubusercontent.com/louielong/blogPic/master/imgkata_banner.png)

> Kata container是一个开源社区，致力于使用轻量级虚拟机构建安全的容器运行时，这些虚拟机感觉和执行起来都像容器，但是使用硬件虚拟化技术作为第二层防御，提供更强的工作负载隔离。

**上一次部署k8s还是去年使用的是v1.17版本，这次打算重新安装一次k8s v1.18版本并搭配kata container试验一下。**

## 二、Docker

这里简要介绍一下docker的组件，为后续的kata containe做一个铺垫。

docker在 1.11 之 后，被拆分成了多个组件以适应 OCI 标准。拆分之后，其包括 docker daemon，  containerd，containerd-shim 以及 runC。组件 containerd 负责集群节点上容器 的生命周期管理，并向上为  docker daemon 提供 gRPC 接口。

<img src="https://raw.githubusercontent.com/louielong/blogPic/master/imgdocker_call_stack.png" alt="docker 架构" style="zoom:33%;" />

1. dockerd 是docker-containerd 的父进程， docker-containerd 是n个docker-containerd-shim 的父进程。
2. Containerd 是一个 gRPC 的服务器。它会在接到 docker daemon 的远程请 求之后，新建一个线程去处理这次请求。依靠 runC 去创建容器进程。而在容器启动之后， runC 进程会退出。
3. runC 命令，是 libcontainer 的一个简单的封装。这个工具可以 用来管理单个容器，比如容器创建，或者容器删除。

container-shim，shim的翻译是垫片，就是修自行车的时候，用来夹在螺丝和螺母之间的小铁片。关于shim本身，网上介绍的文章很少，但是作者在 [Google Groups 里有解释到shim的作用](https://groups.google.com/forum/#!topic/docker-dev/zaZFlvIx1_k)[2]：

- 允许runc在创建&运行容器之后退出
- 用shim作为容器的父进程，而不是直接用containerd作为容器的父进程，是为了防止这种情况：当containerd挂掉的时候，shim还在，因此可以保证容器打开的文件描述符不会被关掉
- 依靠shim来收集&报告容器的退出状态，这样就不需要containerd来wait子进程

关于docker组件的详细介绍这里不再赘述了，想深入了解可以查看[2]。主要关注的是`runc`，后续的kata containe也主要是替换`runc`为Hyper的`runv`。

## 三、kata container

Kata container使用的是`Rust`语言编写，特性如下：

- 安全：运行在专用的内核中，提供网络、I/O和内存隔离，可以利用虚拟化VT扩展中的硬件强制隔离。

- 兼容：支持行业标准，包括OCI容器格式、Kubernetes CRI接口以及传统虚拟化技术。

- 性能：提供与标准Linux容器一致的性能;增强了隔离性，没有标准虚拟机的性能负担。

- 简单：消除了在完整的虚拟机内部嵌套容器的要求； 标准接口使插入和入门变得容易。

kata container基于轻量级虚拟机的容器，不同容器跑在一个个不同的虚拟机（kernel）上，比起传统容器提供了更好的隔离性和安全性，同时继承了容器快速启动和快速部署等优点。这也是本次想使用kata 的原因，能够提供比原生docker更好的隔离性，**Kata 容器每个容器都使用专有内核，避免容器逃逸后影响宿主机的内核**。

<img src="https://raw.githubusercontent.com/louielong/blogPic/master/imgthumbnail.jpg" alt="kata container与VM的区别" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/louielong/blogPic/master/imgkatacontainers_architecture_diagram.jpg" style="zoom: 33%;" />

废话不多说，开始体验一下kata

### 3.1  安装kata

这里使用ubuntu 18.04安装kata最新的版本`Kata Container 1.12.0`参考kata官方安装手册[3]

```shell
admin@k8smaster:~$ ARCH=$(arch)
admin@k8smaster:~$ BRANCH="${BRANCH:-master}"
admin@k8smaster:~$ sudo sh -c "echo 'deb http://download.opensuse.org/repositories/home:/katacontainers:/releases:/${ARCH}:/${BRANCH}/xUbuntu_$(lsb_release -rs)/ /' > /etc/apt/sources.list.d/kata-containers.list"
xdnsadmin@k8smaster:~$ curl -sL  http://download.opensuse.org/repositories/home:/katacontainers:/releases:/${ARCH}:/${BRANCH}/xUbuntu_$(lsb_release -rs)/Release.key | sudo apt-key add -
OK
xdnsadmin@k8smaster:~$ sudo -E apt-get update && sudo -E apt-get -y install kata-runtime kata-proxy kata-shim
```

安装完成后测试一下

```shell
admin@k8smaster:~$ kata-runtime --version
kata-runtime  : 1.12.0-alpha0
   commit   : <<unknown>>
   OCI specs: 1.0.1-dev
```

### 3.2 配置docker的runtime

为了保持安装的docker能够使用，这里默认仍然使用`runc`，但是增加`kata-runtime`做为可选的docker runtime。相关配置可参考官网[kata install](https://github.com/kata-containers/kata-containers/tree/2.0-dev/docs/install)。

```shell
admin@k8smaster:~$ sudo mkdir -p /etc/systemd/system/docker.service.d/
admin@k8smaster:~$  cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/kata-containers.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -D --default-runtime runc --add-runtime kata-runtime=/usr/bin/kata-runtime
EOF
admin@k8smaster:~$ sudo systemctl daemon-reload
admin@k8smaster:~$ sudo systemctl restart docker
```

下面创建两个容器，一个使用 kata runtime，另一个使用默认的 runc。创建完容器之后，查看一下 kata container 的一些信息。

```shell
admin@k8smaster:~$ docker run -d --name kata-test -p 8080:80 --runtime kata-runtime httpd:alpine
ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7
admin@k8smaster:~$ curl http://localhost:8080
<html><body><h1>It works!</h1></body></html>
admin@k8smaster:~$ docker run -d --name runc-test -p 8082:80  httpd:alpine
c7aff63e2d239ea0d27641398f0d3f108999ba3a125c6cef9f1330ccb75c93d4
admin@k8smaster:~$ docker ps -a
CONTAINER ID        IMAGE               COMMAND              CREATED              STATUS              PORTS                  NAMES
c7aff63e2d23        httpd:alpine        "httpd-foreground"   3 seconds ago        Up 2 seconds        0.0.0.0:8082->80/tcp   runc-test
ddbd72466c4c        httpd:alpine        "httpd-foreground"   About a minute ago   Up About a minute   0.0.0.0:8080->80/tcp   kata-test

sadmin@k8smaster:~$ ps -ef | grep docker | grep runtime | grep -v dockerd
root     31346  1035  0 08:30 ?        00:00:00 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-kata-runtime
root     32169  1035  0 08:31 ?        00:00:00 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/c7aff63e2d239ea0d27641398f0d3f108999ba3a125c6cef9f1330ccb75c93d4 -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc

root@k8smaster:/home/xdnsadmin# kata-runtime list    # 需要root权限
ID                                                                 PID         STATUS      BUNDLE                     CREATED                          OWNER
a5468f62e87ea88889d8773247d8b7863f6c8de13fc0a2c6b65ed987fa04db77   23146       running     /run/containers/storage/overlay-containers/a5468f62e87ea88889d8773247d8b7863f6c8de13fc0a2c6b65ed987fa04db77/userdata   2020-07-23T08:14:18.253804592Z   #0
```

查看一下两个容器的内核版本

```shell
admin@k8smaster:~$ docker exec -it kata-test sh
/usr/local/apache2 # uname -a
Linux ddbd72466c4c 5.4.32-49.container #1 SMP Fri Jul 3 05:36:39 UTC 2020 x86_64 Linux
/usr/local/apache2 # cat /proc/version
Linux version 5.4.32-49.container (abuild@lamb04) (gcc version 7.3.0 (Ubuntu 7.3.0-16ubuntu3)) #1 SMP Fri Jul 3 05:36:39 UTC 2020

admin@k8smaster:~$ docker exec -it runc-test sh
/usr/local/apache2 # uname -a    # runc容器内核
Linux c7aff63e2d23 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 Linux

admin@k8smaster:~$ uname -a  #宿主机内核
Linux ubuntu 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

```

查看一下kata容器的虚拟机，可以看到kata专门给容器启动了一个虚拟机沙盒

```shell
admin@k8smaster:~$  ps -ef | grep qemu | grep -v 'grep'
root     31390 31346  0 08:30 ?        00:00:03 /usr/bin/qemu-vanilla-system-x86_64 -name sandbox-ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7 -uuid 0cbfc946-2979-4f44-9d82-62dcdb673c0a -machine pc,accel=kvm,kernel_irqchip,nvdimm -cpu host,pmu=off -qmp unix:/run/vc/vm/ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7/qmp.sock,server,nowait -m 2048M,slots=10,maxmem=8995M -device pci-bridge,bus=pci.0,id=pci-bridge-0,chassis_nr=1,shpc=on,addr=2,romfile= -device virtio-serial-pci,disable-modern=true,id=serial0,romfile= -device virtconsole,chardev=charconsole0,id=console0 -chardev socket,id=charconsole0,path=/run/vc/vm/ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7/console.sock,server,nowait -device nvdimm,id=nv0,memdev=mem0 -object memory-backend-file,id=mem0,mem-path=/usr/share/kata-containers/kata-containers-image_clearlinux_1.12.0-alpha0_agent_e01f289887.img,size=134217728 -device virtio-scsi-pci,id=scsi0,disable-modern=true,romfile= -object rng-random,id=rng0,filename=/dev/urandom -device virtio-rng-pci,rng=rng0,romfile= -device virtserialport,chardev=charch0,id=channel0,name=agent.channel.0 -chardev socket,id=charch0,path=/run/vc/vm/ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7/kata.sock,server,nowait -device virtio-9p-pci,disable-modern=true,fsdev=extra-9p-kataShared,mount_tag=kataShared,romfile= -fsdev local,id=extra-9p-kataShared,path=/run/kata-containers/shared/sandboxes/ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7/shared,security_model=none -netdev tap,id=network-0,vhost=on,vhostfds=3,fds=4 -device driver=virtio-net-pci,netdev=network-0,mac=02:42:ac:11:00:04,disable-modern=true,mq=on,vectors=4,romfile= -rtc base=utc,driftfix=slew,clock=host -global kvm-pit.lost_tick_policy=discard -vga none -no-user-config -nodefaults -nographic -daemonize -object memory-backend-ram,id=dimm1,size=2048M -numa node,memdev=dimm1 -kernel /usr/share/kata-containers/vmlinuz-5.4.32.80-49.container -append tsc=reliable no_timer_check rcupdate.rcu_expedited=1 i8042.direct=1 i8042.dumbkbd=1 i8042.nopnp=1 i8042.noaux=1 noreplace-smp reboot=k console=hvc0 console=hvc1 cryptomgr.notests net.ifnames=0 pci=lastbus=0 iommu=off root=/dev/pmem0p1 rootflags=dax,data=ordered,errors=remount-ro ro rootfstype=ext4 quiet systemd.show_status=false panic=1 nr_cpus=4 agent.use_vsock=false systemd.unit=kata-containers.target systemd.mask=systemd-networkd.service systemd.mask=systemd-networkd.socket scsi_mod.scan=none -pidfile /run/vc/vm/ddbd72466c4ce862891538a8252a83450435b25f488cce98c9f8f9a8e51711e7/pid -smp 1,cores=1,threads=1,sockets=4,maxcpus=4
```

### 3.3 配置k8s与kata的集成

kata 不直接与 k8s 通信，因为对于 k8s 来说，它只跟实现了 CRI 接口的容器管理进程打交道，比如  docker-engine，rkt, containerd(使用 cri plugin) 或 CRI-O，而 kata 跟 runc  是同一个级别的进程。所以如果要与 k8s 集成，则需要安装 CRI-O 或 CRI-containerd 来支持 CRI 接口，本文使用  CRI-O。CRI-O 的 O 的意思是 OCI-compatible，即 CRI-O 是实现 CRI 接口来跑 OCI 容器的[4]。

由于ubuntu 18.04 的[suse源](http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_18.04/)中含有相关软件的包，这里展示如何直接用apt快速安装体验，相关软件的版本如表3.1所示

表3.1  相关软件版本

| 软件名                    | 版本        |
| ------------------------- | ----------- |
| kubelet、kubeadm、kubectl | 1.18.6      |
| cri-o                     | 1.17        |
| kata                      | 1.12-alpha0 |


#### 3.3.1 安装 cri-o

CRI-O - OCI-based implementation of Kubernetes Container Runtime Interface
这是[cri-o](https://github.com/cri-o/cri-o)的github标题，符合OCI基准实现的Kubernetes容器运行时接口，从这个标题很容易看出，这是一个专门服务k8s的容器实现。

- CRI Container Runtime  Interface这是k8s提出的一个概念，在容器界除了最出名的docker和本文介绍的cri-o，还有很多不同的容器实现，例如contrainerd, frakti，rkt等，这些容器实现各有特色，只要支持CRI就可以被k8s支持
- OCI Open Container  Initiative这是开放容器标准，也叫容器runtime，简单来说只要符合这个标准运行的就是容器，例如runC，Kata，gVisor这些runtime创建出来的都是OCI标准容器容器标准，也叫容器runtime，简单来说只要符合这个标准运行的，就是容器，例如runC，Kata，gVisor这些runtime创建出来的都是OCI标准容器

![cri-o logo](https://raw.githubusercontent.com/louielong/blogPic/master/imgcrio-logo.svg)

1）安装cri-o

```shell
admin@k8smaster:~$ CRIO_VERSION=1.17   # 当前ubuntu只有1.17
admin@k8smaster:~$ . /etc/os-release
admin@k8smaster:~$ sudo sh -c "echo 'deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/x${NAME}_${VERSION_ID}/ /' >/etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list"
admin@k8smaster:~$ wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/x${NAME}_${VERSION_ID}/Release.key -O- | sudo apt-key add -
admin@k8smaster:~$ sudo apt-get update -qq
admin@k8smaster:~$ sudo apt-get install cri-o-${CRIO_VERSION}
```

2）修改crio配置文件`/etc/crio/crio.conf`，增加kata runtime

```shell
admin@k8smaster:~$ sudo vim /etc/crio/crio.conf

# 设置kata-runtime为容器运行时
[crio.runtime.runtimes.kata-runtime]
runtime_path = "/usr/bin/kata-runtime"
runtime_type = "oci"
runtime_root = ""

# 由于k8s.gcr.io 无法访问的原因修改镜像源
pause_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2"

# 设置crio镜像下载地址
registries = ["docker.io"]
```

3）修改crio systemd配置，设置自启动

```shell
admin@k8smaster:~$ sudo vim /usr/lib/systemd/system/crio.service

[Service]
....
Restart=on-failure
RestartSec=5

admin@k8smaster:~$ sudo systemctl daemon-reload
admin@k8smaster:~$ sudo systemctl restart crio
admin@k8smaster:~$ sudo service crio status
[sudo] password for louie:
● crio.service - Container Runtime Interface for OCI (CRI-O)
   Loaded: loaded (/usr/lib/systemd/system/crio.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-07-23 02:13:39 UTC; 6h ago
     Docs: https://github.com/cri-o/cri-o
 Main PID: 1876 (crio)
    Tasks: 21
   CGroup: /system.slice/crio.service
           └─1876 /usr/bin/crio
```

4）测试crio

```shell
admin@k8smaster:~$ cat /etc/crictl.yaml
runtime-endpoint: unix:///var/run/crio/crio.sock
image-endpoint: unix:///var/run/crio/crio.sock
timeout: 10

admin@k8smaster:~$ sudo crictl version
Version:  0.1.0
RuntimeName:  cri-o
RuntimeVersion:  1.17.4
RuntimeApiVersion:  v1alpha1
```

#### 3.3.2 安装k8s

1）添加k8s源并安装k8s

```bash
# 使得 apt 支持 ssl 传输
admin@k8smaster:~$ sudo apt-get update && apt-get install -y apt-transport-https
# 下载 gpg 密钥
admin@k8smaster:~$ curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
# 添加 k8s 镜像源
admin@k8smaster:~$ sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
# 更新源列表
admin@k8smaster:~$ sudo apt-get update
# 下载 kubectl，kubeadm以及 kubelet
admin@k8smaster:~$ sudo apt-get install -y kubelet=1.18.6-00 kubeadm=1.18.6-00 kubectl=1.18.6-00
```

2）配置k8s使用crio

根据[官方手册](https://github.com/cri-o/cri-o/blob/master/tutorials/kubeadm.md)，通过cri-o运行k8s需要修改相应文件

```shell
admin@k8smaster:~$ sudo cat /etc/default/kubelet
KUBELET_EXTRA_ARGS=--feature-gates="AllAlpha=false,RunAsGroup=true" --container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock' --runtime-request-timeout=5m
```

修改kubeadm配置，修改cgroup驱动为systemd

```shell
admin@k8smaster:~$ sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

[Service]
....
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --cgroup-driver=systemd --runtime-request-timeout=15m --container-runtime-endpoint=unix:///var/run/crio/crio.sock"
```

重启kubelet

```shell
admin@k8smaster:~$ sudo systemctl daemon-reload; systemctl restart kubelet
```

3）初始化k8s集群

这里通过制定k8s容器仓库为阿里云镜像仓库避免被墙而无法下载镜像，由于阿里云的镜像与官方不同步，因此这里镜像的版本指定为`v1.18.0`，稍微与kubeadm的镜像版本`v1.18.6`不一致。

```shell
admin@k8smaster:~$ sudo kubeadm init --image-repository=registry.aliyuncs.com/google_containers --pod-network-cidr=192.168.0.0/16 --kubernetes-version=v1.18.0 --apiserver-advertise-address=10.37.129.3 --cri-socket=/var/run/crio/crio.sock --v=5
....

admin@k8smaster:~$ mkdir -p $HOME/.kube; cp -i /etc/kubernetes/admin.conf $HOME/.kube/config; chown $(id -u):$(id -g) $HOME/.kube/config
```

- `--pod-network-cidr=192.168.0.0/16`，若于主机所在网络冲突可修改

- `--apiserver-advertise-address=10.37.129.3` ，可不指定API地址，本次安装为了避免网络影响，给机器新增了一块网口



------

***【NOTE】***

1）想查看k8s对应镜像版本可以通过如下命令查看

```shell
admin@k8smaster:~$ kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.18.6
k8s.gcr.io/kube-controller-manager:v1.18.6
k8s.gcr.io/kube-scheduler:v1.18.6
k8s.gcr.io/kube-proxy:v1.18.6
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
```

对于离线情况下，亦可手动下载镜像再导入，这里推荐一个博客[Docker/Kubernetes镜像](https://www.cnblogs.com/kcxg/p/11457209.html)，详细介绍了如何获取`gcr.io`镜像

2）手动安装相关软件最新版本可参考[Katacontainers 与 Docker 和 Kubernetes 的集成](https://lingxiankong.github.io/2018-07-20-katacontainer-docker-k8s.html)，文中相关软件版本较老，这里列出相关软件的最新realease版本

- [Runc release](https://github.com/opencontainers/runc/releases)

- [Crictl release](https://github.com/kubernetes-sigs/cri-tools/releases)

- [conmon](https://github.com/containers/conmon)

- [CNI plugins](https://github.com/containernetworking/plugins/releases)

------



#### 3.3.3 配置k8s集群网络

这里选用calico网络，参考[calico官网](https://docs.projectcalico.org/getting-started/kubernetes/quickstart)

1）安装Tigera Calico操作符和自定义资源定义

```shell
admin@k8smaster:~$ kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```

2）通过创建必要的自定义资源来安装Calico

```shell
admin@k8smaster:~$ kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

这里需要注意，若k8s集群初始化时`CIDR`指定的不是`192.168.0.0/16`需要修改`custom-resources.yaml`文件中的`CIDR`地址

3）确认所有pod都在运行

```shell
admin@k8smaster:~$ watch kubectl get pods -n calico-system
```

等待所有pod的状态为`running`

4）本次安装只使用了一个节点，需要配置master节点“无污点”

```shell
admin@k8smaster:~$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/<your-hostname> untainted
```

5）查看所有master节点状态

```shell
admin@k8smaster:~$ kubectl get nodes -o wide
NAME     STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
ubuntu   Ready    master   6h36m   v1.18.6   10.211.55.5   <none>        Ubuntu 18.04.4 LTS   4.15.0-112-generic   cri-o://1.17.4
```

6）查看所有pod状态

```shell
admin@k8smaster:~$ kubectl get pods -A -o wide
NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE     IP                NODE     NOMINATED NODE   READINESS GATES
calico-system     calico-kube-controllers-5687f44fd5-49d4d   1/1     Running   0          6h34m   192.168.243.194   ubuntu   <none>           <none>
calico-system     calico-node-grtnk                          1/1     Running   0          6h34m   10.211.55.5       ubuntu   <none>           <none>
calico-system     calico-typha-7cd5478c69-vldhp              1/1     Running   0          6h34m   10.211.55.5       ubuntu   <none>           <none>
kube-system       coredns-7ff77c879f-7b7lc                   1/1     Running   0          6h36m   192.168.243.192   ubuntu   <none>           <none>
kube-system       coredns-7ff77c879f-m8glx                   1/1     Running   0          6h36m   192.168.243.195   ubuntu   <none>           <none>
kube-system       etcd-ubuntu                                1/1     Running   0          6h36m   10.211.55.5       ubuntu   <none>           <none>
kube-system       kube-apiserver-ubuntu                      1/1     Running   0          6h36m   10.211.55.5       ubuntu   <none>           <none>
kube-system       kube-controller-manager-ubuntu             1/1     Running   0          6h36m   10.211.55.5       ubuntu   <none>           <none>
kube-system       kube-proxy-mzpk5                           1/1     Running   0          6h36m   10.211.55.5       ubuntu   <none>           <none>
kube-system       kube-scheduler-ubuntu                      1/1     Running   0          6h36m   10.211.55.5       ubuntu   <none>           <none>
tigera-operator   tigera-operator-6659cdcd96-7nzsq           1/1     Running   0          6h35m   10.211.55.5       ubuntu   <none>           <none>
```

#### 3.3.4 测试k8s使用kata runtime

k8s从1.14版本开始设置了[runt imeclass](https://kubernetes.io/zh/docs/concepts/containers/runtime-class/) ，同时crio从1.14版本开始增加RuntimeHandler以取代[trusted/untrusted runtime](https://lingxiankong.github.io/2018-07-20-katacontainer-docker-k8s.html)，因此k8s使用kata不能再使用如下配置了。

```yaml
annotations:
        io.kubernetes.cri-o.TrustedSandbox: "false"
```

1）首先创建kata runtime class

编写kata runtime class配置文件
```yaml
# k8s-runtimecleass.yaml

apiVersion: node.k8s.io/v1beta1  # RuntimeClass is defined in the node.k8s.io API group
kind: RuntimeClass
metadata:
  name: kata
handler: kata-runtime  # handler 名称需要与/etc/crio/crio.conf配置一致
```

创建kata runtime class
```shell
admin@k8smaster:~$ kubectl apply -f k8s-runtimecleass.yaml
admin@k8smaster:~$ kubectl get runtimeclass
NAME   HANDLER        AGE
kata   kata-runtime   1m
```

2）创建kata runtime pod

```shell
admin@k8smaster:~$ cat kata-test.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-kata
  labels:
    app: kata
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kata
  template:
    metadata:
      labels:
        app: kata
    spec:
      containers:
      - name: test-kata
        image: lingxiankong/alpine-test
        ports:
        - containerPort: 8080
      runtimeClassName: kata  # 注明使用kata runtime
---
kind: Service
apiVersion: v1
metadata:
  name: test-kata
spec:
  selector:
    app: kata
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080

admin@k8smaster:~$ kubectl apply -f kata-test.yaml
```

等待kata pod 启动，查看pod状态

``` shell
admin@k8smaster:~$ kubectl get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP                NODE     NOMINATED NODE   READINESS GATES
test-kata-64b4c55f9c-bh7xl   1/1     Running   0          25h   192.168.243.199   ubuntu   <none>           <none>
test-kata-64b4c55f9c-cjvg7   1/1     Running   0          25h   192.168.243.198   ubuntu   <none>           <none>

admin@k8smaster:~$ kubectl get pod test-kata
admin@k8smaster:~$ kubectl get deployment test-kata
admin@k8smaster:~$ kubectl get svc test-kata
```







【参考链接】

1）[Docker组件介绍（二）：shim, docker-init和docker-proxy](https://jiajunhuang.com/articles/2018_12_24-docker_components_part2.md.html)

2）[Docker组件介绍（一）：runc和containerd](https://jiajunhuang.com/articles/2018_12_22-docker_components.md.html)

3）[Install Kata Containers on Ubuntu](https://github.com/kata-containers/documentation/blob/master/install/ubuntu-installation-guide.md)

4）[Katacontainers 与 Docker 和 Kubernetes 的集成](https://lingxiankong.github.io/2018-07-20-katacontainer-docker-k8s.html)


























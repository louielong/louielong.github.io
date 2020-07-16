---
title: 【转载】扩展 Kubernetes 之 CRI
date: 2020-07-16 10:49:52
tags:
 - k8s
keywords:
 - k8s
 - CRI
categories:
 - k8s
description:
 - 扩展 Kubernetes 之 CRI
summary_img:
 - https://raw.githubusercontent.com/louielong/blogPic/master/img20200716105458.png
---

【转载】本文转载自[扩展 Kubernetes 之 CRI](https://cloud.tencent.com/developer/article/1579900)

最近在研究k8s搭配kata container，对于其中的CRI很是困惑，借由这篇文章学习一下CRI及其相关概念。

## 一、简介

### 1.1 CRI 是什么

容器运行时插件（Container Runtime Interface，简称 CRI）是 Kubernetes v1.5 引入的容器运行时接口，它将 Kubelet 与容器运行时解耦，将原来完全面向 Pod 级别的内部接口拆分成面向 Sandbox 和 Container 的 gRPC 接口，并将镜像管理和容器管理分离到不同的服务。

CRI 主要定义了两个 grpc interface.
- `RuntimeService`：容器(container) 和 (Pod)Sandbox 运行时管理
- `ImageService`：拉取、查看、和移除镜像

OCI (开放容器标准): 定义了 ImageSpec（镜像格式, 比如文件夹结构，压缩方式）和 RuntimeSpec（如何运行，比如支持 create, start, stop, delete）。代表实现有：runC，Kata（以及它的前身 runV 和 Clear Containers），gVisor。

CRI 区别于 OCI，CRI的定义比较简单直接，只是定义了一套协议（grpc 接口）。代表实现有 kubernetest 内置的 dockershim, CRI-containerd（或者 containerd with CRI plugin）, cri-o

### 1.2 CRI 位于什么位置

在 kubernetes 中:

![k8belet与CRI](https://raw.githubusercontent.com/louielong/blogPic/master/imgxlyb0th1c7.png)

![CRI组成](https://raw.githubusercontent.com/louielong/blogPic/master/imgrchsiecsii.png)

在和OCI，调度层的角度看:

```js
graph LR
OrchestrationAPI --> ContainerAPI-criRuntime
ContainerAPI-criRuntime --> KernelAPI-ociRuntime
```

## 二、CRI/CRI Runtime

### 2.1 CRI Runtime 的执行流程

经典的 kubernetes runtime 执行流程

![k8s runtime 流程](https://raw.githubusercontent.com/louielong/blogPic/master/img00r84irf8c5.png)

- 执行流程里面核心组件是 kubelet/KubeGenericRuntimeManager 他调用很多其他组件，比如 cm (ContainerManager/podContainerManager/cgroupManager/cpuManager/deviceManager, pod 级别的资源管理), RuntimeService (grpc 调用 CRI 的客户端) 共同完成 SyncPod 的操作。

经典的 dockershim -> containerd 的流程 (称为 docker cri)

![k8s调用docker流程](https://raw.githubusercontent.com/louielong/blogPic/master/imgcczv8nowa.png)

1. Kubelet 通过 CRI 接口（gRPC）调用 dockershim，请求创建一个容器。CRI 即容器运行时接口（Container Runtime Interface），这一步中，Kubelet 可以视作一个简单的 CRI Client，而 dockershim 就是接收请求的 Server。目前 dockershim 的代码其实是内嵌在 Kubelet 中的，所以接收调用的凑巧就是 Kubelet 进程
2. dockershim 收到请求后，转化成 Docker Daemon 能听懂的请求，发到 Docker Daemon 上请求创建一个容器。
3. Docker Daemon 早在 1.12 版本中就已经将针对容器的操作移到另一个守护进程——containerd 中了，因此 Docker Daemon 仍然不能帮我们创建容器，而是要请求 containerd 创建一个容器；
4. containerd 收到请求后，并不会自己直接去操作容器，而是创建一个叫做 containerd-shim 的进程，让 containerd-shim 去操作容器。这是因为容器进程需要一个父进程来做诸如收集状态，维持 stdin 等 fd 打开等工作。而假如这个父进程就是 containerd，那每次 containerd 挂掉或升级，整个宿主机上所有的容器都得退出了。而引入了 containerd-shim 就规避了这个问题（containerd 和 shim 并不是父子进程关系）；
5. 我们知道创建容器需要做一些设置 namespaces 和 cgroups，挂载 root filesystem 等等操作，而这些事该怎么做已经有了公开的规范了，那就是 OCI（Open Container Initiative，开放容器标准）。它的一个参考实现叫做 runC。于是，containerd-shim 在这一步需要调用 runC 这个命令行工具，来启动容器；
6. runC 启动完容器后本身会直接退出，containerd-shim 则会成为容器进程的父进程，负责收集容器进程的状态，上报给 containerd，并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理，确保不会出现僵尸进程。

直接对接 cri-containerd/cri-o 的运行时

![k8s对接containerd](https://raw.githubusercontent.com/louielong/blogPic/master/img7z2bq3l54g.png)

![k8s对接cri-o](https://raw.githubusercontent.com/louielong/blogPic/master/imgv9xrgk5bht.png)

使用 cri-containerd 的调用流程更为简洁, 省去了上面的调用流程的 1，2 两步

常见 CRI runtime 实现

| CRI容器运行时  | 维护者     | 主要特性           | 容器引擎                 |
| -------------- | ---------- | ------------------ | ------------------------ |
| Dockershim     | Kubernetes | 内置实现、特性最新 | docker                   |
| cri-o          | Kubernetes | OCI标准            | OCI(runc、kaata、gVisor) |
| cri-containerd | Containerd | 基于containerd     | OCI(runc、kaata、gVisor) |
| Frakti         | Kubernetes | 虚拟化容器         | hyperd、docker           |
| rktlet         | Kubernetes | 支持rkt            | rkt                      |
| PouchContainer | Alibaba    | 富容器             | OCI(runc、kata)          |
| Virlet         | Mirantis   | 虚拟机和QCOW2镜像  | Libvirt(KVM)             |

Cri-containerd

![img](https://raw.githubusercontent.com/louielong/blogPic/master/imgztykbbu2la.png)

执行流程为:

1. Kubelet 通过 CRI runtime service API 调用 cri plugin 创建 pod
2. cri 通过 CNI 创建 pod 的网络配置和 namespace
3. cri 使用 containerd 创建并启动 pause container (sandbox container) 并且把这个 container 置于 pod 的 cgroups/namespace
4. Kubelet 接着通过 CRI image service API 调用 cri plugin, 获取容器镜像
5. cri 通过 containerd 获取容器镜像
6. Kubelet 通过 CRI runtime service API 调用 cri, 在 pod 的空间使用拉取的镜像启动容器
7. cri 通过 containerd 创建/启动 应用容器, 并且把 container 置于 pod 的 cgroups/namespace. Pod 完成启动.





## 【参考文献】

1）[CRI-feisky](https://feisky.gitbooks.io/kubernetes/plugins/CRI.html)

2）[K8S Runtime CRI OCI contained dockershim 理解](https://blog.csdn.net/u011563903/article/details/90743853)

3）[CRI API 定义](https://github.com/kubernetes/cri-api/blob/master/pkg/apis/runtime/v1alpha2/api.proto)

4）[cri-containerd](https://github.com/containerd/cri)

5）[containerd](https://github.com/containerd/containerd)

6）[K8S 容器运行时-feisky](https://zhuanlan.zhihu.com/p/47012368)

7）[critools](https://github.com/kubernetes-sigs/cri-tools)

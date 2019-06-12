---
title: Docker容器上限思考
date: 2019-03-07 14:01:26
tags:
  - docker
keywords:
  - docker
categories:
  - docker
description:
  - Docker容器创建上限的思考
summary_img:
  - https://i.imgur.com/gV5VmE3.png
---

## 一、前言

在研究`Helm`时发现其创建的随机容器名跟docker类似都是左右形式的组合，因此想到Docker随机容器名的上限是多少，会不会因为随机容器名的重复而导致无法再创建随机容器名的容器实例而达到容器的上限。

## 二、随机容器名生成

根据[my2cents](https://frightanic.com/computers/docker-default-container-names/)的`How does Docker generate default container names?`文章介绍，得知docker随机容器名的生成是分为左右值的，即拥有两个独立的数组。查看docker容器名生成的[源代码](https://github.com/moby/moby/blob/master/pkg/namesgenerator/names-generator.go)可以看到左值都是一些形容词，共计100个，右值都是一些`amazing man`，并且给出了wiki百科的介绍页面，共计235位。(统计截止到2019年3月7日)

有意思的是代码的最后一行，当容器名为`boring_wozniak`时会重新生成，[Steve Wozniak](https://en.wikipedia.org/wiki/Steve_Wozniak)是Apple I和Apple II的发明人，看来码代码的人十分推崇Wozniak，加了专门的彩蛋。

```go
if name == "boring_wozniak" /* Steve Wozniak is not boring */ {
  goto begin
}
```

根据容器名规则以及容器名实例不能重复原则使用排列组合可以得知能够创建的随机容器名实例上限是100*235-1=23499个。

**这种容器名生成的优势是什么？**

通过使用**形容词-单词**的组合方式生成的容器名比枯燥的hash数字更为好看也便于记忆。

## 三、容器上限

问题来了，在单一主机中到底能够运行多少个容器？

根据`stack overflow`上的[Is there a maximum number of containers running on a Docker host?](https://stackoverflow.com/questions/21799382/is-there-a-maximum-number-of-containers-running-on-a-docker-host)回答，翻译如下[1]

docker容器创建实例有很多的系统限制，一些重要的信息如下：

- 容器的配置
- 容器里运行的程序
- 系统的内核版本、发行版本、docker版本

**详细的运行限制如下**：

- 链接到虚拟网络适配器docker0网桥的设备限制：(每个网桥最多1023)
- 挂载联合文件系统(AUFS)和shm文件系统：(最大挂载数量1048576)
- 在镜像image上创建的层数layer数量：（最多127layer每个镜像）
- fork出来一个docker-containerd-shim的管理进程:(每个容器平均3M左右，系统最大进程数sysctl kernel.pid_max)
- docker daemon守候进程管理容器的内部数据：(~400k 每个容器)
- 创建内核的*cgroups*和*namespace*
- 打开文件描述符：(启动中的容器16个左右) ulimit -n and sysctl fs.file-max
- 端口映射，-p将会在宿主机上为每一个映射的端口启动一个外部进程：（平均每个端口占用~4.5MB）
- –net=none 和 –net=host参数将有助于减少网络消耗。

**Container 服务**
总的资源消耗取决于你容器内运行的程序，而不是docker本身，如果你在虚拟机中运行应用程序node,ruby,python,java，内存的消耗将是主要问题。
1000个进程会消耗大大量的IO 链接。1000个进程同时运行也会引起大量的上下文交换，

1023 Docker busybox images

```shell
nc -l -p 80 -e echo
```

共计使用约1G的内核空间和3.5G的系统内存

1023 普通进程

```shell
nc -l -p 80 -e echo
```

将会消耗约75MB的内核空间和125MB的系统内存

此外，

连续启动1023个容器预计耗费约8分钟，关闭1023个容器预计耗费约6分钟

### 实验

有文章做了一个实验来启动500个容器，感兴趣的可以查看[docker 启动500个容器测试](https://blog.csdn.net/warrior_0319/article/details/79730191)

## 四、结论

根据以上信息可以得知随机容器名的限制不会是容器创建上限的障碍，容器实例本身所消耗的资源如内存、网络等才是最关键的影响。



【参考链接】

1）[Is there a maximum number of containers running on a Docker host?](https://stackoverflow.com/questions/21799382/is-there-a-maximum-number-of-containers-running-on-a-docker-host)

2）[docker 最大container数量调研](https://blog.csdn.net/warrior_0319/article/details/79716901)

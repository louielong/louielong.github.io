---
title: ubuntu下安装配置NTP服务器
date: 2018-05-10 14:35:22
tags:
  - NTP
  - Ubuntu
keywords:
  - NTP
categories:
  - Ubuntu
description:
  - NTP 服务器安装
summary_img:
---

## 1 前言

在建立Linux主机集群时，为了避免集群主机时间不同以及长时间运行下所导致的时间偏差，需要进行时间同步(synchronize)。Linux系统下，一般使用ntp服务来同步不同机器的时间。NTP（Network Time Protocol）即网络时间协议，就是通过网络协议使计算机之间的时间同步化。

**时间同步方式**

NTP在linux下有两种时钟同步方式，分别为直接同步和平滑同步[1]：

- 直接同步

使用ntpdate命令进行同步，直接进行时间变更。如果服务器上存在一个12点运行的任务，当前服务器时间是13点，但标准时间时11点，使用此命令可能会造成任务重复执行。因此使用ntpdate同步可能会引发风险，因此该命令也多用于配置时钟同步服务时第一次同步时间时使用，如：系统重启时。

- 平滑同步

使用ntpd进行时钟同步，可以保证一个时间不经历两次，它每次同步时间的偏移量不会太陡，是慢慢来的，这正因为这样，ntpd平滑同步可能耗费的时间比较长。对ntp时间同步修正原理感兴趣的可以查看官方文档：[How NTP Works](https://www.eecis.udel.edu/~mills/ntp/html/warp.html)



时钟的跃变，对于某些程序会导致很严重的问题。许多应用程序依赖连续的时钟，取得的时间是线性的，一些操作，例如数据库事务，通常会地依赖这样的事实：时间不会往回跳跃。不幸的是，ntpdate调整时间的方式就是我们所说的”跃变“：在获得一个时间之后，ntpdate使用settimeofday(2)设置系统时间，这有几个非常明显的问题[2]：

第一，这样做不安全。ntpdate的设置依赖于ntp服务器的安全性，攻击者可以利用一些软件设计上的缺陷，拿下ntp服务器并令与其同步的服务器执行某些消耗性的任务。由于ntpdate采用的方式是跳变，跟随它的服务器无法知道是否发生了异常（时间不一样的时候，唯一的办法是以服务器为准）。

第二，这样做不精确。一旦ntp服务器宕机，跟随它的服务器也就会无法同步时间。与此不同，ntpd不仅能够校准计算机的时间，而且能够校准计算机的时钟。

第三，这样做不够优雅。由于是跳变，而不是使时间变快或变慢，依赖时序的程序会出错（例如，如果ntpdate发现你的时间快了，则可能会经历两个相同的时刻，对某些应用而言，这是致命的）。因而，唯一一个可以令时间发生跳变的点，是计算机刚刚启动，但还没有启动很多服务的那个时候。其余的时候，理想的做法是使用ntpd来校准时钟，而不是调整计算机时钟上的时间。

NTPD 在和时间服务器的同步过程中，会把 BIOS 计时器的振荡频率偏差——或者说 Local Clock 的自然漂移(drift)——记录下来。这样即使网络有问题，本机仍然能维持一个相当精确的走时。

## 2 安装ntp

ntp的安装是服务器和客户端集成在一起的，在ubuntu或centos下安装ntp服务器都非常简单，这里以ubuntu为例，使用NTP命令即可：

```shell
sudo apt install -y ntp
```

如果需要ntpdate工具则需要额外安装ntpdate

```shell
sudo apt install -y ntpdate
```

## 3 配置ntp服务器或server

ntp的配置文件存放在`/etc/ntp.conf`，打开文件并进行修改，部分参数说明如下

### 3.1 使用server命令设定上层NTP服务器

设定方式：

```
server [address] [options...]
```

在server后面填写服务器地址（可以使IP或主机名），之后是命令参数主要包括autokey，brust，ibrust，key，minpoll ，maxpoll。这里最长使用的是ibrust和prefer。

| 参数   | 含义                                              |
| ------ | ------------------------------------------------- |
| brust  | 当时间服务器不可达时，将发送间隔为2秒的连续八个包 |
| ibrust | 当时间服务器不可达时，将发送间隔为2秒的连续八个包 |
| prefer | 当其他数据相同时该节点将作为首选时间同步          |

其它参数的详细说明可参考NTP的帮助文档（`man 5 ntp.conf`）。

### 3.2 使用restrict命令管理权限控制

设定方式：

```
restrict [address] mask [mask] [parameter]
```

其中parameter的参数主要有以下这些：

- ignore: 拒绝所有类型的NTP联机；
- nomodify: 客户端不能使用ntpc与ntpq这两个程序来修改服务器的时间参数，但客户端仍可透过这个主机来进行网络校时；
- noquery: 客户端不能使用ntpq，ntpc等指令来查询时间服务器，等于不提供NTP的网络校时；
- notrap: 不提供trap这个远程事件登录(remote event logging)的功能；
- notrust: 拒绝没有认证的客户端；

如果你没有在 parameter 的地方加上任何参数的话，这表示该 IP 或网段不受任何限制。

### 3.3 使用driftfile记录时间差异

设定方式：

```
driftfile [可以被ntpd写入的目录与档案]
```

因为预设的 NTP Server 本身的时间计算是依据 BIOS 的芯片震荡周期频率来计算的，但是这个数值与上层 Time Server 不见得会一致啊！所以 NTP 这个 daemon (ntpd) 会自动的去计算我们自己主机的频率与上层 Time server 的频率，并且将两个频率的误差记录下来，记录下来的档案就是在 driftfile 后面接的完整档名当中了！关于档名你必须要知道：

- driftfile 后面接的档案需要使用完整路径文件名；
- 该档案不能是连结档；
- 该档案需要设定成 ntpd 这个 daemon 可以写入的权限；
- 该档案所记录的数值单位为：百万分之一秒 (ppm)；

driftfile 后面接的档案会被 ntpd 自动更新，所以他的权限一定要能够让 ntpd 写入才行。

### 3.4 使用statsdir和filegen开启统计分析

设定方式：

```
statsdir directory_path
filegen name file filename [type type] [link | nolink] [enable | disable]
```

当打开统计分析是会在directory_path目录下产生filegen中所设定的统计文件。

### 3.5 指定接口

ntp服务开启时默认监听所有的接口，如果想指定ip段或者接口则按照一下方式

```
interface [listen | ignore | drop] [all | ipv4 | ipv6 | wildcard | name | address[/prefixlen]]
```

listen指定监听接口，ignore忽略该接口，drop丢弃接口的请求数据包，接口类型可以指定为v4、v6或接口名。

### 3.6 配置文件示例

为了支持IPv6这里添加清华源的NTP服务器，清华源官方链接：[传送门](https://tuna.moe/help/ntp/)

配置文件完整示例如下：
```
# /etc/ntp.conf, configuration for ntpd; see ntp.conf(5) for help
# 时间差异文件
driftfile /var/lib/ntp/ntp.drift

# 接口设置
interface listen eth0
interface ignore ipv4

# 分析统计信息
#statsdir /var/log/ntpstats/

statistics loopstats peerstats clockstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable

# 上层ntp server.
pool ntp1.aliyun.com iburst
pool ntp2.aliyun.com iburst
pool ntp3.aliyun.com iburst
pool ntp4.aliyun.com iburst
# 清华源提供IPv4和IPv6双栈
pool ntp.tuna.tsinghua.edu.cn iburst perfer

# Use Ubuntu's ntp server as a fallback.
pool ntp.ubuntu.com

# 不允许来自公网上ipv4和ipv6客户端的访问
restrict -4 default kod notrap nomodify nopeer noquery limited
restrict -6 default kod notrap nomodify nopeer noquery limited
# 准许以下网络的ntp请求
restrict -6 240c:6100:ffff:: netmask 64 nomodify
restrict 172.16.0.1 netmask 20 nomodify

# 让NTP Server和其自身保持同步，如果在/etc/ntp.conf中定义的server都不可用时，将使用local时间作为ntp服务提供给ntp客户端.
restrict 127.0.0.1
restrict ::1

# Needed for adding pool entries
restrict source notrap nomodify noquery

# If you want to provide time to your local subnet, change the next line.
# (Again, the address is an example only.)
#broadcast 192.168.123.255

# If you want to listen to time broadcasts on your local subnet, de-comment the
# next lines.  Please do this only if you trust everybody on the network!
#disable auth
#broadcastclient

#Changes recquired to use pps synchonisation as explained in documentation:
#http://www.ntp.org/ntpfaq/NTP-s-config-adv.htm#AEN3918

#server 127.127.8.1 mode 135 prefer    # Meinberg GPS167 with PPS
#fudge 127.127.8.1 time1 0.0042        # relative to PPS for my hardware

#server 127.127.22.1                   # ATOM(PPS)
#fudge 127.127.22.1 flag3 1            # enable PPS API
```

## 4 启动ntp服务

启动ntp服务

```shell
sudo service ntp restart
```

通过ntpq命令查看ntp同步状态

```shell
watch ntpq -p
Every 2.0s: ntpq -p                                                                                                                                 

     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 ntp1.aliyun.com .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp2.aliyun.com .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp3.aliyun.com .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp4.aliyun.com .POOL.          16 p    -   64    0    0.000    0.000   0.000
 ntp.ubuntu.com  .POOL.          16 p    -   64    0    0.000    0.000   0.000
+2001:67c:1560:8 192.53.103.108   2 u  411  512  377  401.548  -19.089   4.396
*2001:67c:1560:8 17.253.34.125    2 u  512  512  377  379.682    0.776  18.600

```



上述字段的含义如下：

- remote: 指的就是本地机器所连接的远程NTP服务器；
- refid: 指的是给远程服务器提供时间同步的服务器；
- st: 远程服务器的层级别（stratum）. 由于NTP是层型结构,有顶端的服务器,多层的Relay Server再到客户端。所以服务器从高到低级别可以设定为1-16. 为了减缓负荷和网络堵塞,原则上应该避免直接连接到级别为1的服务器的；
- when: 几秒钟前曾经做过时间同步化更新的动作；
- poll: 本地机和远程服务器多少时间进行一次同步(单位为秒).
  在一开始运行NTP的时候这个poll值会比较小,那样和服务器同步的频率也就增加了,可以尽快调整到正确的时间范围.之后poll值会逐渐增大,同步的频率也就会相应减小；
- reach: 已经向上层 NTP 服务器要求更新的次数；
- delay: 网络传输过程当中延迟的时间，单位为 10^(-6) 秒；
- offset: 时间补偿的结果，单位与 10^(-3) 秒；
- jitter: Linux 系统时间与 BIOS 硬件时间的差异时间， 单位为 10^(-6) 秒。简单地说这个数值的绝对值越小我们和服务器的时间就越精确；
- *: 它告诉我们远端的服务器已经被确认为我们的主NTP Server,我们系统的时间将由这台机器所提供；
- +: 它将作为辅助的NTP Server和带有*号的服务器一起为我们提供同步服务. 当*号服务器不可用时它就可以接管；
- -: 远程服务器被clustering algorithm认为是不合格的NTP Server；
- x: 远程服务器不可用；

**ntpdate 强制同步时间**

如果本机与时间同步服务器时间间隔太大可以通过ntpdate命令来进行同步，需要指出的是ntpdate使用前需要停止ntpd的进程。

```shell
sudo ntpdate ntp1.ailiyun.com
```

【Note】

ntp监听的是123端口，部分操作系统或防火墙等可能限制123端口因此需要放开端口限制。







【参考链接】

1)[centos 7 ntp 同步说明](https://www.linuxidc.com/Linux/2015-11/124911.htm)

2)[NTP 安装与配置](https://www.cnblogs.com/kerrycode/archive/2015/08/20/4744804.html)


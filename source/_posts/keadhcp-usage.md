---
title: KEA DHCP安装配置及使用
date: 2018-04-07 12:26:15
tags:
  - KEA
  - ubuntu
keywords:
  - DHCP
categories:
  - ubuntu
description:
  - KEA DHCP的安装配置及使用
summary_img:
---

## 1 前言

![KEAD DHCP LOGO](https://www.isc.org/wp-content/uploads/2017/04/Kea-logo-300x250.png)

KEA 是由Internet Systems Consortium开发的开源DHCPv4 / DHCPv6服务器。 Kea是一款高性能，可扩展的DHCP服务器引擎，可以轻松修改和扩展钩子库。KEA具有以下特性[1]：

- 开源，使用MPL 2.0许可证
- 直接地址分配支持v4和v6，或DHCPv6 前缀授权
- 动态地址分配和主机地址保留
- 更新DNS记录作为续租或过期的动态DNS
- MAC地址追踪，包括 v4和v6
- 支持自定义扩展钩子库


## 2 安装KEA

KEA的安装可以通过apt的方式直接安装，但是如果需要数据库支持或者需要使用钩子扩展则需要自行编译，本文以自编译的方式介绍如何安装配置KEA。

1) 首先下载源码：

```shell
root@ubuntu:~# wget https://ftp.isc.org/isc/kea/1.3.0/kea-1.3.0.tar.gz
```

2) 安装必要的编译软件包

```shell
root@ubuntu:~# apt install -y gcc build-essential make libmysql++-dev openssl libssl-dev libboost-system-dev liblog4cplus-dev liblog4cplus-1.1-9 libmysqlclient-dev
```

3) 配置编译

指明需要使用mysql，若需要修改默认安装路径需要单独指定`--prefix`和`--exec-prefix`两个参数，前者是编译生成的二进制文件拷贝路径，后者是软件运行时依赖库的查找路径，可以通过`./configure -h`查看。

```shell
root@ubuntu:~# tar xf kea-1.3.0.tar.gz
root@ubuntu:~# cd kea-1.3.0
root@ubuntu:~/kea-1.3.0# ./configure  --with-dhcp-mysql=/usr/bin/mysql_config
```

4) 编译

配置完编译依旧是编译二连`make`和`make install`

```shell
root@ubuntu:~# make -j8
root@ubuntu:~# make install
```

编译完成后默认安装到`/usr/local/kea`目录下，相应的配置文件放置在`/usr/local/kea/etc/kea`路径下。

## 3 配置KEA

kea的配置文件是json格式，配置完kea可以先使用[json在线解析](https://www.bejson.com/jsoneditoronline/)查看配置是否正确，需要注意的是必须去除配置中的注释才能正确解析。详细的kea配置需要查看官网的介绍[2]，同时配置文件`kea-dhcp6.conf.sample`示例中也有详细的解释。


如下所示为配置文件参数的含义[3]：

```json
{
"Dhcp6": {
    // 向服务器租用地址时间，以秒为单位
    "valid-lifetime": 4000,
    // 可选，续借时间，如果没有指定将根据RFC 2131进行设置
    "renew-timer": 1000,
    // 可选，重新绑定时间，如果没有指定这将根据RFC 2131进行设置
    "rebind-timer": 2000,

    "interfaces-config": {
        // 1. 指定服务器要监听哪张网卡的DHCP消息，可以指定多张网卡。
        // 2.允许使用*，如："interfaces": ["*"]，表示监听所有网卡
        "interfaces": ["eth0"],
        // 默认raw，表示处理所有报文
        // udp：处理udp报文
        "dhcp-socket-type": "udp"，
        // 只有dhcp-socket-type为udp才生效
        // 默认是same-as-inbound：从哪里来滚哪里去
        // use-routing：从哪里来，滚哪里去，得问下kernel的路由表(routing table)       "outbound-interface": "use-routing"
    },
    // 控制平面接收
    "control-socket": {
        "socket-type": "unix",
        "socket-name": "/tmp/kea-dhcp6-ctrl.sock"
    },
    // 租期数据使用库类型指定，类型不同，对应的配置也有所不同，这里以MySQL为例
    "lease-database": {
        // 支持memfile", "mysql", "postgresql"， "cql"四个选项
        "type": "mysql",
        // 数据库所在的主机ip
        "host": "localhost",
        // 数据库端口号
        "port": 3306,
        // 数据库名称
        "name": "keadhcp",
        // 数据库用户名
        "user": "root",
        // 数据库密码
        "password": "root",
        // 当type为memfile这里会涉及到一个比较重要的配置，这里不说明，详情请看(http://kea.isc.org/wiki/LFCDesign)
        // 1. 指定服务器将执行租约文件清理（LFC）的时间间隔（以秒为单位
        // 2. 默认3600，0的时候表示禁用lease file cleanup(LFC)
        // "lfc-interval": 1800
    },

    // 1.下面的配置可选。主机预定数据使用的数据库类型。和租期配置类同，不在赘述
    // 2. 当然你也可以不使用数据库，在数据量不大的情况下推荐使用配置文件。随着数据量的增大可以改用数据库
    // 3. 这个配置允许数据库和配置文件共存使用
    // 4. 同时使用时，先检查配置文件，在检查数据库的数据
    // "hosts-database": {
    //     "type": "mysql",
    //     "host": "localhost",
    //     "port": 3306,
    //     "name": "kea",
    //     "user": "kea",
    //     "password": "kea"
    // },

    "subnet6": [{
        // 子网标识符，没有指定或者为0，则自动分配
        // 建议手动分配，如果有多个子网，某个子网被删除，id可能被自动重新分配，导致租期数据混乱
        "id":"1024"
        // 网段 这里需要注意下网段必须和服务器所在网段一样，不然接收不到客户的请求
        "subnet": "2001:db8:1::/64",
        // 可分配地址范围
        "pools": [{"pool": "2001:db8:1::1000 - 2001:db8:1::2000"}]
    }]
}，
// 若不配置logging字段则日志记录默认输出在终端
"Logging":
{
  "loggers": [
    {
        "name": "kea-dhcp6",
        "output_options": [
            {
                // 指定日志输出路径
                "output": "/var/log/dhcp/dhcp6.log",
                // 当为true时每次更新日志文件都会同步到磁盘
                "flush": true,
                // 单个日志文件最大容量
                "maxsize": 1048576,
                // 同时存储日志文件最大个数
                "maxver": 8
            }
        ],
        // 日志输出等级
        "severity": "INFO",
        // 当日志输出等级为debug时，可选择debug输出等级0-100,0最低
        "debuglevel": 0
    }
  ]
}
}
```

配置完成后重启kea服务即可，重启哪些服务可以在`/usr/local/kea/etc/kea/keactrl.conf`文件指定

```shell
root@ubuntu:/usr/local/kea/etc/kea$ cat keactrl.conf
# This is a configuration file for keactrl script which controls
# the startup, shutdown, reconfiguration and gathering the status
# of the Kea's processes.

# prefix holds the location where the Kea is installed.
prefix=/usr/local/kea

# Location of Kea configuration files.
kea_dhcp4_config_file=${prefix}/etc/kea/kea-dhcp4.conf
kea_dhcp6_config_file=${prefix}/etc/kea/kea-dhcp6.conf
kea_dhcp_ddns_config_file=${prefix}/etc/kea/kea-dhcp-ddns.conf
kea_ctrl_agent_config_file=${prefix}/etc/kea/kea-ctrl-agent.conf

# Location of Kea binaries.
exec_prefix=/usr/local/kea
dhcp4_srv=${exec_prefix}/sbin/kea-dhcp4
dhcp6_srv=${exec_prefix}/sbin/kea-dhcp6
dhcp_ddns_srv=${exec_prefix}/sbin/kea-dhcp-ddns
ctrl_agent_srv=${exec_prefix}/sbin/kea-ctrl-agent

# Start DHCPv4 server?
dhcp4=yes

# Start DHCPv6 server?
dhcp6=yes

# Start DHCP DDNS server?
dhcp_ddns=no

# Start Control Agent?
ctrl_agent=yes

# Be verbose?
kea_verbose=no
```

重启DHCP服务

```shell
root@ubuntu:~# keactrl start
```

若需要指定重启v6或v4则需要添加相应参数

```shell
root@ubuntu:~# keactrl start -s dhcpv6
```

关于如何配置KEA的钩子模式可以查看：[传送门](https://github.com/zorun/kea-hook-runscript)

## 4 其他

### 4.1 KEA 高可用

目前KEA 1.3版本尚不支持HA高可用模式，官方介绍将在[1.4版本](https://www.isc.org/blogs/planning-for-kea-1-4/)支持，当前状态下若想使用HA可以通过数据库后端HA的方式来实现，也可以通过keepalived来实现，见[传送门](https://arch-ed.dk/kea-1-3-0-site-failover)，结合上一篇的keepalived可以很好的实现，原文中需要三台服务器，笔者在实验中使用了两台服务器也可以测试通过，由于keepalived不支持监听UDP端口，因此主要的实现方式是添加keepalived健康检查脚本定时检查kea进程。

### 4.2 KEA配合phpIPAM

当前phpIPAM并没有直接的插件配合KEA，因此需要自己实现，实现方式有两种：1)直接采用数据库同步的方式，将kea的数据导入phpIPAM中；2）采用phpIPAM的restful接口，同时phpIPAM也给出了API客户端[4]。遇到的困难是KEA的子网标记ID和phIPAM中子网号ID同步转换出错进而导致hosts同步错误，尤其是当KEA的子网号在重启会重新分配(未指定子网号时)或修改子网但未更新子网号时造成的租期数据混乱。



【参考链接】

1)[KEA官网](https://www.isc.org/kea/)

2)[KEA官方文档](https://ftp.isc.org/isc/kea/1.3.0/doc/kea-guide.html)

3)[KEA配置介绍](https://blog.csdn.net/z475382220/article/details/78844227)

4)[phpIPAM API客户端](https://github.com/phpipam/phpipam-api-clients)

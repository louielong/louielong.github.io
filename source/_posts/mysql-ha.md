---
title: MySQL双主复制 + keepalived故障转移实现
tags:
  - mysql
  - keepalived
  - ubuntu
keywords:
  - mysql
categories:
  - ubuntu
description:
  - MySQL主主复制配置
summary_img:
top:
---

## 1 前言

MySQL复制中较常见的复制架构有“一主一从”、“一主多从”、“双主”、“多级复制”和“多主环形机构”等，在项目实施中遇到需要进行故障转移的需求：两台服务器每台都安装MySQL，当一个MySQL服务器故障时另一个MySQL服务器能够继续提供服务，这要求两个MySQL之间能够进行数据复制同时需要监控两台服务器的状态。

本次使用MySQL的双主复制以及keepalived的HA机制来实现。

## 2 环境准备

两台服务器：
服务器MySQL-HA-1(主) 192.168.10.101
服务器MySQL-HA-2(主) 192.168.10.102
虚拟服务节点IP       192.168.10.103
Mysql版本：mysql  Ver 14.14 Distrib 5.7.21
System OS：ubuntu 16.04



## 3 mysql数据库配置

### 3.1 mysql数据库介绍

![mysql](https://gss2.bdstatic.com/9fo3dSag_xI4khGkpoWK1HF6hhy/baike/w%3D268%3Bg%3D0/sign=e35e494a6159252da3171a020ca06406/ac6eddc451da81cb037c289d5366d016082431c3.jpg)

MySQL是一个[**关系型数据库管理系统**](https://baike.baidu.com/item/%E5%85%B3%E7%B3%BB%E5%9E%8B%E6%95%B0%E6%8D%AE%E5%BA%93%E7%AE%A1%E7%90%86%E7%B3%BB%E7%BB%9F)**，**由瑞典MySQL AB 公司开发，目前属于 [Oracle](https://baike.baidu.com/item/Oracle) 旗下产品。MySQL 是最流行的[关系型数据库管理系统](https://baike.baidu.com/item/%E5%85%B3%E7%B3%BB%E5%9E%8B%E6%95%B0%E6%8D%AE%E5%BA%93%E7%AE%A1%E7%90%86%E7%B3%BB%E7%BB%9F)之一，在 WEB 应用方面，MySQL是最好的 RDBMS (Relational Database Management System，关系数据库管理系统) 应用软件。

MySQL是一种关系数据库管理系统，关系数据库将数据保存在不同的表中，而不是将所有数据放在一个大仓库内，这样就增加了速度并提高了灵活性。

MySQL所使用的 SQL 语言是用于访问[数据库](https://baike.baidu.com/item/%E6%95%B0%E6%8D%AE%E5%BA%93)的最常用标准化语言[1]。

### 3.1 安装数据库

```shell
apt install -y mysql-server
```

【Tips】：安装过程需要输入数据库密码，出于后续部署的自动化考虑，希望自动化部署中不被打断，解决这个问题有两种方法：

1) 预先设置密码

使用`debconf-set-selections`工具预先将密码写入

```shell
sudo debconf-set-selections <<< 'mysql-server mysql-server/root_password password your_password'
sudo debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password your_password'
sudo apt-get -y install mysql-server
```

将`your_password`替换为想要设置的mysql root账户密码，针对不同的mysql版本会有相应的改变，参见[传送门](https://stackoverflow.com/questions/7739645/install-mysql-on-ubuntu-without-a-password-prompt)

2)静默安装

使用如下命令通过非交互的方式静默安装mysql

```shell
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mysql-server
```

在安装完成后mysql是没有密码的，root用户下是可以免密进入命令行的，然后再修改mysql的root用户访问密码

```shell
mysql -uroot -e"SET PASSWORD FOR 'root'@'localhost' = PASSWORD('passwd');"
```

将`passwd`替换为自己的密码即可。

3.2 数据库初始化配置

在修改配置前最好先备份mysql配置文件

```shell
cp /etc/mysql/mysql.conf.d/mysqld.cnf  /etc/mysql/mysql.conf.d/mysqld.cnf-bak
```

1）设置时区

在`mysqld.cnf`的`[mysqld]`后加入`default-time-zone = '+8:00'`

```shell
sed -i "/^\[mysqld\]/a\default-time-zone = \'+8:00\'" /etc/mysql/mysql.conf.d/mysqld.cnf
```

### 3.3 修改数据库配置

设置需要同步的数据库，此处以数据库`test`为例

1）修改MySQL-HA-1服务器数据库配置

主要修改的地方如下

```shell
bind-address		= ::    # 指定允许数据库访问的IP，"::"表明允许v4和v6访问

server-id		= 1                     # 指定mysql的编号 该编号作为数据库集群的识别号因此不能冲突
log_bin			= /var/log/mysql/mysql-bin.log   # 开启二进制log文件
binlog_format           = mixed
relay-log               = relay-bin
relay-log-index         = slave-relay-bin.index
auto-increment-offset   = 2     # 设置自增长偏移，这里有两个数据库因此每次增长为2
auto-increment-increment = 1    # 设置自增长起始
binlog_do_db            = test  # 设置需要同步的数据库名称
binlog_do_db            = test1
```

完整的配置文件如下，这里将原配置文件的注释屏蔽掉


```shell
root@HA-1:~# cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep -v '#'

[mysqld_safe]
socket		= /var/run/mysqld/mysqld.sock
nice		= 0

[mysqld]
user		= mysql
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
port		= 3306
basedir		= /usr
datadir		= /var/lib/mysql
tmpdir		= /tmp
lc-messages-dir	= /usr/share/mysql
skip-external-locking
bind-address		= ::
key_buffer_size		= 16M
max_allowed_packet	= 16M
thread_stack		= 192K
thread_cache_size       = 8
myisam-recover-options  = BACKUP
query_cache_limit	= 1M
query_cache_size        = 16M
general_log_file        = /var/log/mysql/mysql.log
general_log             = 1
log_error = /var/log/mysql/error.log
server-id		= 1
log_bin			= /var/log/mysql/mysql-bin.log
binlog_format           = mixed
expire_logs_days        = 10
max_binlog_size         = 100M
relay-log               = relay-bin
relay-log-index         = slave-relay-bin.index
auto-increment-offset   = 2
auto-increment-increment = 1
binlog_do_db            = test
binlog_do_db            = test1
```

其他参数含义可参考[mysql配置文件详解](http://www.jb51.net/article/48082.htm)


2）修改MySQL-HA-2服务器数据库配置

修改HA-2节点的数据库配置文件，大致内容一致，只是在`server-id`和`auto-increment-increment`需要修改。

```shell
root@HA-2:~# cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep -v '#'
[mysqld_safe]
socket		= /var/run/mysqld/mysqld.sock
nice		= 0

[mysqld]
user		= mysql
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
port		= 3306
basedir		= /usr
datadir		= /var/lib/mysql
tmpdir		= /tmp
lc-messages-dir	= /usr/share/mysql
skip-external-locking
bind-address		= ::
key_buffer_size		= 16M
max_allowed_packet	= 16M
thread_stack		= 192K
thread_cache_size       = 8
myisam-recover-options  = BACKUP
query_cache_limit	= 1M
query_cache_size        = 16M
general_log_file        = /var/log/mysql/mysql.log
general_log             = 1
log_error = /var/log/mysql/error.log
server-id		= 2            # server id设置为2
log_bin			= /var/log/mysql/mysql-bin.log
binlog_format           = mixed
expire_logs_days        = 10
max_binlog_size         = 100M
relay-log               = relay-bin
relay-log-index         = slave-relay-bin.index
auto-increment-offset   = 2
auto-increment-increment = 2    # 增长起始设置为2
binlog_do_db            = test
binlog_do_db            = test1
```

修改完成后**两个节点**的数据库都需要重启

```shell
service mysql restart
```

### 3.3 设置主从数据库

1)将HA-1设置为HA-2的主数据库

首先在HA-1节点数据库创建同步账户

```shell
root@HA-1:~# mysql -uroot -p
mysql> GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO sync@'192.168.10.102' IDENTIFIED BY 'sync';
mysql> flush privileges;
```

随后查看数据库状态信息

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |    465 | test,test1    |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

记录下`File`和`Position`两个参数值，这是数据库主从同步的关键，也是告诉从数据自那哪个起点开始同步

在**HA-2**节点的数据库输入以下信息，切记是在HA-2节点输入

```mysql
mysql> change master to master_host='192.168.10.101',master_user='sync',master_password='sync',master_log_file='mysql-bin.000001',master_log_pos=465;
```

随后重启HA-2节点的数据库

```shell
root@HA-2:~# service mysql restart
```

然后在HA-2节点查看mysql的slave信息，确保下述两个值为yes

> Slave_IO_Running:Yes
> Slave_SQL_Running:Yes

```shell
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.10.101
                  Master_User: copyuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 465
               Relay_Log_File: relay-bin.00002
                Relay_Log_Pos: 146
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
```

2)将HA-2设置为HA-1的主数据库

设置过程与上一步一致，首先创建同步账户随后添加主数据库信息，需要注意的是IP地址的修改

在HA-2上创建同步账户

```shell
root@HA-2:~# mysql -uroot -p
mysql> GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO sync@'192.168.10.101' IDENTIFIED BY 'sync';
mysql> flush privileges;
```

随后查看数据库状态信息

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000002 |    465 | test,test1    |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

记录下`File`和`Position`两个参数值，这是数据库主从同步的关键，也是告诉从数据自那哪个起点开始同步

在**HA-1**节点的数据库输入以下信息，切记是在HA-1节点输入

```mysql
mysql> change master to master_host='192.168.10.102',master_user='sync',master_password='sync',master_log_file='mysql-bin.000001',master_log_pos=465;
```

随后重启HA-1节点的数据库

```shell
root@HA-1:~# service mysql restart
```

然后在HA-1节点查看mysql的slave信息，确保下述两个值为yes

> Slave_IO_Running:Yes
> Slave_SQL_Running:Yes

```shell
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.10.102
                  Master_User: copyuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 465
               Relay_Log_File: relay-bin.00003
                Relay_Log_Pos: 146
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
```

### 3.4 测试数据库同步

在HA-1节点创建一个数据库`test`

```mysql
mysql> create database test;
```

查看HA-2主机是否同步了HA-1上的数据变化

```mysql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| test          |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

可以看出HA-2节点的数据库同步了HA-1节点的数据库，在配置成双主复制后任一节点数据库发生改变另一节点数据库都会进行同步。

在配置完成数据库后若想对数据库进行访问只能访问单一节点数据库的IP，如果希望访问一个固定IP让数据库并能够实现故障自动切换就需要配合keepalived或者HAproxy进行代理。

## 4 keepalived 安装

### 4.1 keepalived介绍

![keepalived](http://www.keepalived.org/images/ka-header.png)

Keepalived软件起初是专为LVS负载均衡软件设计的，用来管理并监控LVS集群系统中各个服务节点的状态，后来又加入了可以实现高可用的VRRP功能。因此，Keepalived除了能够管理LVS软件外，还可以作为其他服务（例如：Nginx、Haproxy、MySQL等）的高可用解决方案软件。

Keepalived软件主要是通过VRRP协议实现高可用功能的。VRRP是Virtual Router RedundancyProtocol(虚拟路由器冗余协议）的缩写，VRRP出现的目的就是为了解决静态路由单点故障问题的，它能够保证当个别节点宕机时，整个网络可以不间断地运行。

所以，Keepalived 一方面具有配置管理LVS的功能，同时还具有对LVS下面节点进行健康检查的功能，另一方面也可实现系统网络服务的高可用功能。

这里借用博客[4]的有关keepalived的集群工作原理示意图

![keepalived状态切换示意图](https://i.imgur.com/iZyFaCC.png)

Keepalived高可用对之间是通过 VRRP进行通信的， VRRP是遑过竞选机制来确定主备的，主的优先级高于备，因此，工作时主会优先获得所有的资源，备节点处于等待状态，当主挂了的时候，备节点就会接管主节点的资源，然后顶替主节点对外提供服务。

在 Keepalived服务对之间，只有作为主的服务器会一直发送 VRRP广播包,告诉备它还活着，此时备不会枪占主，当主不可用时，即备监听不到主发送的广播包时，就会启动相关服务接管资源，保证业务的连续性.接管速度最快可以小于1秒。

### 4.2 keepalived的安装

安装方式分为两种：apt直接安装和手动编译安装

1)手动编译安装

手动编译的好处是可以使用较新的源码，首先下载源码

```shell
wget http://www.keepalived.org/software/keepalived-1.4.2.tar.gz
```

安装必要的编译包

```shell
apt-get install -y gcc build-essential make curl libssl-dev libnl-3-dev libnl-genl-3-dev libsnmp-dev
```

配置编译，prefix指明需要安装在哪里，也可以不配置使用默认路径

```shell
tar xf keepalived-1.4.2.tar.gz
cd keepalived-1.4.2
./configure --prefix=/usr/local/keepalived
```

配置完成后直接编译二连`make`和`make install`即可

2) apt安装

```shell
apt install keepalived
```

### 4.3 keepapiled配置文件

keepalived服务安装完成之后，后面的主要工作就是在keepalived.conf文件中配置HA和负载均衡。一个功能比较完整的常用的keepalived配置文件，主要包含三块：*全局定义块*、*VRRP实例定义块*和*虚拟服务器定义块*。全局定义块是必须的，如果keepalived只用来做ha，虚拟服务器是可选的。下面数据库HA的配置文件模板：

#### 4.3.1 keepalived.conf配置

**HA-1主机**上的keepalived.conf文件的修改：

```shell
root@HA-1:~# cat /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
    router_id HA-1
}

vrrp_script chk_mysql {
    script /etc/keepalived/bin/chk_mysql.sh    #健康监测脚本路径
    interval 2
    fall 3
    rise 2
}

vrrp_instance VI_MYSQL {
    state MASTER
    interface enp0s9       # 监听网卡
    virtual_router_id 100  # 虚拟路由编号，同一实例可以一致，但是其权重一定不能一致
    nopreempt
    priority 100           # 权重，两个节点不能一样
    advert_int 1
    virtual_ipaddress {
        192.168.10.103      # 虚拟IP地址
    }
    notify /etc/keepalived/bin/kpad_notify.sh     # keep状态传入脚本，通过该脚本可得知当前keep运行状态
    track_script {
        chk_mysql            # 健康检查配置
    }
}

}
```
##### 4.3.1.2 健康监测脚本

创建脚本存放目录

```shell
root@HA-1:~# mkdir -p /etc/keepalived/bin
```

1)keepalived状态脚本

创建脚本`/etc/keepalived/bin/kpad_notify.sh`内容如下

```bash
root@HA-1:~# cat /etc/keepalived/bin/kpad_notify.sh

#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

log_file="/var/log/test/keepalived/keepalived.log"

log() {
echo "$(date +"%Y-%m-%d %H:%M:%S.%4N") [$STATE] $1" >> $log_file
}

case $STATE in
    "MASTER")
        echo 'MASTER' > /tmp/keepalived-state
        exit 0
        ;;
    "BACKUP")
        echo 'BACKUP' > /tmp/keepalived-state
        exit 0
        ;;
    "FAULT")
        echo 'FAULT' > /tmp/keepalived-state
        log "keepalived status is fault."
        exit 0
        ;;
    *)
        log "unknown keepalived status."
        exit 1
        ;;
esac
```

设置脚本运行权限

```shell
root@HA-1:~# chmod +x /etc/keepalived/bin/kpad_notify.sh
```

2）配置mysql健康检查脚本

编辑`/etc/keepalived/bin/chk_mysql.sh`脚本内容如下，脚本的大致思路是如果在master和backup状态下mysqld进程不存在则尝试重启mysql，若重启失败则任务该节点的mysql彻底故障，进行故障转移。

```bash
#!/bin/bash
#########################################################################
# File Name: chk_mysql.sh
# Author: louie.long
# Mail: ylong@biigroup.cn
# Created Time: Wed 04 Apr 2018 10:44:20 AM CST
# Description: check mysql service
#########################################################################

STATE=`cat /tmp/keepalived-state`
log_file="/var/log/test/keepalived/keepalived.log"
service_name="mysqld"
service_cmd="/etc/init.d/mysql"
get_pid=`pidof $service_name`

log() {
echo "$(date +"%Y-%m-%d %H:%M:%S.%4N") [$STATE] $1" >> $log_file
}

case $STATE in
    "MASTER")
        if [ "${get_pid}" == "" ]; then
            log "$service_name service isn't exist."
            log "Try to restart $service_name service."
            $service_cmd start
            if [ $? -eq 0 ]; then
                log "restart $service_name service successfully."
            else
                log "restart $service_name service failed."
                exit 1
            fi
        fi
        exit 0
        ;;
    "BACKUP")
        if [ "${get_pid}" == "" ]; then
            log "$service_name service isn't exist."
            log "Try to restart $service_name service."
            $service_cmd start
            if [ $? -eq 0 ]; then
                log "restart $service_name service successfully."
            else
                log "restart $service_name service failed."
                exit 1
            fi
        fi
        exit 0
        ;;
    "FAULT")
        exit 0
        ;;
       *)
        exit 1
        ;;
esac
```

然后执行

```shell
root@HA-1:~# chmod +x /etc/keepalived/bin/chk_mysql.sh
```

随后重启keepalived服务

```shell
root@HA-1:~# service keepalived restart
```

#### 4.3.2 在HA-2节点上配置

基本配置个HA-1节点上一样，两个健康监测脚本完全一致，不同的是keepalived.conf脚本中权重值和节点初始属性

```shell
root@HA-2:~# cat /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
    router_id HA-1
}

vrrp_script chk_mysql {
    script /etc/keepalived/bin/chk_mysql.sh    #健康监测脚本路径
    interval 2
    fall 3
    rise 2
}

vrrp_instance VI_MYSQL {
    state BACKUP
    interface enp0s9       # 监听网卡
    virtual_router_id 100  # 虚拟路由编号，同一实例可以一致，但是其权重一定不能一致
    nopreempt
    priority 90           # 权重，两个节点不能一样
    advert_int 1
    virtual_ipaddress {
        192.168.10.103      # 虚拟IP地址
    }
    notify /etc/keepalived/bin/kpad_notify.sh     # keep状态传入脚本，通过该脚本可得知当前keep运行状态
    track_script {
        chk_mysql            # 健康检查配置
    }
}

}
```

在拷贝`/etc/keepalived/bin/kpad_notify.sh`和`/etc/keepalived/bin/chk_mysql.sh `两个脚本后重启keepalived服务

```shell
root@HA-2:~# service keepalived restart
```

#### 4.3.3 测试

在HA-1和HA-2分别执行ip addr show dev enp0s9命令查看HA-1和HA-2对VIP（群集虚拟IP）的控制权。HA-1主的查看结果：

```shell
root@HA-1:~# ip addr show dev enp0s9
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:fd:98:1b brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.101/20 brd 192.168.15.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet 192.168.10.103/32 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 240c:f:1:4000:20c:29ff:fefd:981b/64 scope global mngtmpaddr dynamic
       valid_lft 2591546sec preferred_lft 604346sec
    inet6 fe80::20c:29ff:fefd:981b/64 scope link
       valid_lft forever preferred_lft forever
```

可以看到生成了192.168.10.101这个虚拟IP。

停止HA-1的keepalived服务，HA-2将会成为新的主节点，HA-2主的查看结果：

```shell
root@HA-2:~# ip addr show dev enp0s9
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:fd:98:1b brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.102/20 brd 192.168.15.255 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet 192.168.10.103/32 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 240c:f:1:4000:20c:29ff:fefd:981b/64 scope global mngtmpaddr dynamic
       valid_lft 2591855sec preferred_lft 604655sec
    inet6 fe80::20c:29ff:fefd:981b/64 scope link
       valid_lft forever preferred_lft forever
```

可以看到生成了192.168.10.103这个虚IP。

MySQL远程登录测试：

```shell
root@HA-2:~#  mysql -h192.168.10.103 -uroot -p
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2119
Server version: 5.7.21-0ubuntu0.16.04.1-log (Ubuntu)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 2     |
+---------------+-------+
1 row in set (0.00 sec)

```

说明在客户端访问VIP地址，由HA-2主机提供响应的，当前状态下HA-2充当主服务器。

【Note】

经过测试在纯IPv6的环境下上述HA依然可以正常运行。







【参考链接】

1)[mysql介绍](https://baike.baidu.com/item/mySQL/471251?fr=aladdin)

2)[mysql安装跳过密码设置](https://stackoverflow.com/questions/7739645/install-mysql-on-ubuntu-without-a-password-prompt)

3)[keepalived官网](http://www.keepalived.org)

4)[keepaliced介绍](https://www.cnblogs.com/clsn/p/8052649.html)




















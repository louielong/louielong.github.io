---
title: IP地址管理(IPAM)
date: 2018-03-30 18:23:56
tags:
  - IPAM
  - ubuntu
keywords:
  - IPAM
categories:
  - ubuntu
description:
  - IP地址管理环境搭建
summary_img:
---



## 1 前言

IPAM：IP Address Management  ，及IP地址管理，用于发现、监视、审核和管理企业网络上使用的 IP 地址空间。目前开源的IPAM有许多，根据维基百科[IPAM](https://en.wikipedia.org/wiki/IP_address_management)介绍有十多种IPAM类软件，在研究了一下活跃的项目后发现phpIPAM相当的不错，尝试把玩了一下。

![phpipam](https://phpipam.net/css/images/old/phpipam_logo_small.png)

## 2 快速搭建

phpIPAM的官方有一个demo可以访问：https://demo.phpipam.net/login/，登录的用户名和密码为：Admin / ipamadmin，在首页也可以查看。

查找了下发现docker hub中有相关的镜像[1]，直接从docker上拉取一个镜像作为快速搭建体验。

### 2.1 拉取相关镜像

安装phpIPAM需要mysql数据库作为存储，首先获取mysql数据库

```shell
root@ubuntu:~# docker pull mysql:5.6
```

启动mysql数据库，将`my-secret-pw`换成自己的数据库密码，同时需要映射数据库存储路径`/my_dir/phpipam`

```shell
root@ubuntu:~# docker run --name phpipam-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -v /my_dir/phpipam:/var/lib/mysql -d mysql:5.6
```

获取phpIPAM的镜像

```shell
root@ubuntu:~# docker pull pierrecdn/phpipam
```

启动phpIPAM镜像，需要将docker镜像内的80端口映射到本机的80端口

```shell
root@ubuntu:~# docker run -ti -d -p 80:80 --name ipam --link phpipam-mysql:mysql pierrecdn/phpipam
```

### 2.2 phpIPAM配置

随后登录http://<HOST_IP>:80即可以访问phpIPAM的页面，首次登录需要安装数据库，主要是导入phpIPAM的数据库表

1)自动导入数据库

点击“Automatic database installtion”

![phpIPAM自动导入数据库](https://cloud.githubusercontent.com/assets/4225738/8746785/01758b9e-2c8d-11e5-8643-7f5862c75efe.png)

2)输入数据库密码

输入2.1节中设置的数据库密码，随后点击“Install phpipam database”

![输入数据库密码](https://cloud.githubusercontent.com/assets/4225738/8746789/0ad367e2-2c8d-11e5-80bb-f5093801e139.png)

3)重新设置访问密码

输入新密码后点击"Save setting"，重新登录即可进入页面

![重置访问密码](https://cloud.githubusercontent.com/assets/4225738/8746790/0c434bf6-2c8d-11e5-9ae7-b7d1021b7aa0.png)

一个phpIPAM的环境就快速搭建起来了，可以随意操作玩耍了。

## 3 完整搭建

如果想完整的搭建一个phpIPAM在本地同时做一些自定义的修改就需要下载源码并搭建，过程也不复杂官方文档也有详细描述[2]。

phpIPAM当前仅支持IPv4的主机扫描，并不支持IPv6的主机扫描，对于该项需求在github上已经有人提出了但是当前仍没有解决[3]。

接下来以ubuntu16.04为例讲解如何搭建phpIPAM环境。

### 3.1 安装必要软件包

```shell
root@ubuntu:~# apt-get install apache2 mariadb-server php php-pear php7.0-gmp php7.0-mysql php7.0-mbstring php7.0-gd php7.0-mcrypt php7.0-curl git cron
```

安装过程需要配置mysql root用户密码，安装完成后拉取phpIPAM的代码

```shell
root@ubuntu:~# cd /var/www/html/
root@ubuntu:/var/www/html# git clone https://github.com/phpipam/phpipam.git .
root@ubuntu:/var/www/html# git checkout 1.3.1
```

### 3.2 配置相应软件

1) 配置mysql数据库

启动mysql(MariaDB) 数据库，并配置mysql的root账户

```shell
root@ubuntu:~# /etc/init.d/mysql restart
root@ubuntu:~# mysql_secure_installation
NOTE: RUNNING ALL PARTS OF THIS SCRIPT ISRECOMMENDED FOR ALL MariaDB

      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!
In order to log into MariaDB to secure it,we'll need the current

password for the root user.  If you've just installed MariaDB, and

you haven't set the root password yet, thepassword will be blank,

so you should just press enter here.
Enter current password for root (enter fornone): 

OK, successfully used password, movingon...

Setting the root password ensures thatnobody can log into the MariaDB

root user without the proper authorisation.
Set root password? [Y/n] y

New password: 

Re-enter new password: 

Password updated successfully!

Reloading privilege tables..

 ...Success!
By default, a MariaDB installation has ananonymous user, allowing anyone

to log into MariaDB without having to havea user account created for

them. This is intended only for testing, and to make the installation

go a bit smoother.  You should remove them before moving into a

production environment.
Remove anonymous users? [Y/n] y

 ...Success!
Normally, root should only be allowed toconnect from 'localhost'.  This

ensures that someone cannot guess at theroot password from the network.
Disallow root login remotely? [Y/n] n

 ...skipping.
By default, MariaDB comes with a databasenamed 'test' that anyone can

access. This is also intended only for testing, and should be removed

before moving into a productionenvironment.

Remove test database and access to it?[Y/n] y

 -Dropping test database...

 ...Success!

 -Removing privileges on test database...

 ...Success!

Reloading the privilege tables will ensurethat all changes made so far

will take effect immediately.

Reload privilege tables now? [Y/n] y

 ...Success!

Cleaning up...

All done! If you've completed all of the above steps, your MariaDB

installation should now be secure.

Thanks for using MariaDB!
```

2) 配置apache代理

配置`/etc/apache2/apache2.conf`

```
<Directory "/var/www/html">
	Options FollowSymLinks
	AllowOverride all
	Order allow,deny
	Allow from all
</Directory>
```

设置apache时区

```shell
root@ubuntu:~# grep timezone /etc/php/7.0/apache2/php.ini
; Defines the default timezone used by the date functions
; http://php.net/date.timezone
date.timezone = Asia/Shanghai
```

使能apache重写

```shell
root@ubuntu:~# a2enmod rewrite
```

重启apapche2服务

```shell
root@ubuntu:~# /etc/init.d/apache2 start
```

3) 配置phpIPAM

首先拷贝配置文件

```shell
root@ubuntu:/var/www/html# cp config.dist.php config.php
```

修改配置文件的数据库信息

```php
$db['host'] = 'localhost';
$db['user'] = "root";
$db['pass'] = "r00tme";
$db['name'] = 'phpipam';
$db['port'] = 3306;
```

配置ipam的v4主机定时检查

```shell
# update host statuses exery 15minutes"    
*/15 * * * * /usr/bin/php/opt/cfiec/6dnsnetm/ipam/functions/scripts/pingCheck.php";
*/15 * * * * /usr/bin/php/opt/cfiec/6dnsnetm/ipam/functions/scripts/discoveryCheck.php";
```

配置ipam数据库每天备份和自动删除10天以上的数据备份

```shell
@daily /usr/bin/mysqldump -uroot -pr00tmephpipam > /opt/cfiec/6dnsnetm/ipam/db/bkp/phpipam_bkp_$(date+"%y%m%d").db";
@daily /usr/bin/find/opt/cfiec/6dnsnetm/ipam/db/bkp/ -ctime +10 -exec rm {} \;
```

可以通过crontab -l 查看配置是否正确

重启cron服务

```shell
root@ubuntu:/var/www/html# /etc/init.d/cron restart
```

随后打开网页访问相应的IP地址即可进入到安装界面。





【参考链接】

1)[phpIPAM docker 主页](https://hub.docker.com/r/clinta/phpipam/)

2)[phpIPAM官网](https://phpipam.net/documents/installation/)

3)[phpIPAM IPv6主机扫描](https://github.com/phpipam/phpipam/issues/777)






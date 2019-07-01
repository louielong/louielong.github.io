---
title: Compass4nfv 安装 OPNFV Gambia
date: 2019-01-23 15:00:29
tags:
  - OPNFV
  - compass4nfv
keywords:
  - compass4nfv
categories:
  - OPNFV
description:
  - 使用compass4nfv安装opnfv Gambia版本
summary_img:
  - https://raw.githubusercontent.com/louielong/blogPic/master/imgToX4rXC.png
---

## 一 前言

OPNFV社区保持着每半年发布一个大版本的劲头，在2018年11月发布了第7个版本 Gambia，在观察社区中各个安装软件后，由于compass项目会支持较多的部署策略及新特性[1]，如k8s等。此外，compass也是少有的*支持最少一个计算节点和一个控制节点*的OPNFV部署工具。本次尝试使用compass4nfv来部署OPNFV的Gambia版本。在部署过程中也遇到了很多的坑，同样的官方的部署指导文档总会在很自然的忽略掉一些很关键的点，这里选择记录一下部署安装以及排错的过程。

Ps：compass4nfv的部署真的是一波N多折，修复一个问题又出现另一个问题，时不时的网络下载失败让人崩溃至极。

![fix bug](https://raw.githubusercontent.com/louielong/blogPic/master/imgFix_bug.gif)

## 二 软件准备

### 2.1 克隆代码

首先需要克隆compass4nfv的代码，链接地址如下

```shell
git clone https://gerrit.opnfv.org/gerrit/compass4nfv
```

随后切换到Gambia的稳定分支版本

```shell
git checkout -b stable/gambia
```

### 2.2 离线部署软件镜像准备

Compass4nfv支持离线部署模式，离线部署需要预先下载以下镜像系统，链接为：[https://artifacts.opnfv.org/compass4nfv.html](https://artifacts.opnfv.org/compass4nfv.html)

然后选定相应的版本镜像即可，这里选择[compass4nfv/gambia/opnfv-2018-11-19_08-25-04.tar.gz](https://artifacts.opnfv.org/compass4nfv/gambia/opnfv-2018-11-19_08-25-04.tar.gz)

准备好以上安装文件后，修改部署配置文件

1) 部署脚本配置

`compass4nfv/deploy.sh`中指明镜像路径

```shell
export TAR_URL=file:///home/opnfv/opnfv-2018-11-19_08-25-04.tar.gz
```

接下来就要进行相应网络及节点的配置。

## 三 安装部署

![水滴特效](https://raw.githubusercontent.com/louielong/blogPic/master/imgGhF8HQE.gif)

不同于Fuel的MAAS和saltstack部署，Compass使用的是cobbler和ansible部署，不了解[cobbler](http://cobbler.github.io/)的可以查阅下相关资料。不同版本的安装可能会稍有不同，请以官方指导手册为准。

### 3.1 节点配置

部署中可以参考代码仓库中默认的其他节点网络及相应配置`compass4nfv/deploy/conf/hardware_environment`

下面分别详细介绍网络配置`network.yml`和部署策略`os-nosdn-nofeature-ha.yml`

#### 3.1.1  网络配置文件

部署中的节点网络配置都在此文件进行描述，实际的硬件网络拓扑可以参看先前的Fuel部署，依旧是第一块网卡作为PXE，第二块网卡作为External网络，第三块网卡作为私有网络通信，承载存储、管理、虚机间通信等，需要注意的是以上的所有的网卡都可以合一，即最少一块网卡也能部署。下面的网络配置字面意思也很好理解，这里不做过多介绍了，详细介绍参考官方[Configure network](https://opnfv-compass4nfv.readthedocs.io/en/stable-gambia/release/installation/configure-network.html)，PXE网络设置为`10.20.0.0/24`（后续安装中发现管理网络也在这个网段），外网设置为`192.168.20.0/24`，需要额外指明的是租户网络和存储网络配置的有vlan_tag，必须在交换机上配置好。

```shell
opnfv@ubuntu:~/compass4nfv/deploy/conf/hardware_environment/bii-pod1$ cat network.yml

---
nic_mappings: []
bond_mappings: []

provider_net_mappings:
  - name: br-provider #必须为此名否则后续安装ovs配置网桥会报错
    network: physnet
    interface: eth1
    type: ovs
    role:
      - controller

sys_intf_mappings:
  # PXE Network
  - name: mgmt
    interface: eth0
    type: normal
    vlan_tag: None
    role:
      - controller
      - compute

  - name: tenant
    interface: eth2
    type: normal
    vlan_tag: 101
    role:
      - controller
      - compute

  - name: storage
    interface: eth2
    type: normal
    vlan_tag: 102
    role:
      - controller
      - compute

  - name: external
    interface: eth1
    type: normal
    vlan_tag: None
    role:
      - controller
      - compute

ip_settings:
  - name: mgmt
    ip_ranges:
      - - "10.20.0.50"
        - "10.20.0.100"
    dhcp_ranges:
      - - "10.20.0.10"
        - "10.20.0.49"
    cidr: "10.20.0.0/24"
    gw: "10.20.0.1"
    role:
      - controller
      - compute

  - name: tenant
    ip_ranges:
      - - "192.168.102.1"
        - "192.168.102.50"
    cidr: "192.168.102.0/24"
    role:
      - controller
      - compute

  - name: storage
    ip_ranges:
      - - "192.168.103.1"
        - "192.168.103.50"
    cidr: "192.168.103.0/24"
    role:
      - controller
      - compute

  - name: external
    ip_ranges:
      - - "192.168.20.10"
        - "192.168.20.50"
    cidr: "192.168.20.0/24"
    gw: "192.168.20.1"
    role:
      - controller
      - compute

internal_vip:
  ip: 10.20.0.100
  netmask: "24"
  interface: mgmt

public_vip:
  ip: 192.168.20.103
  netmask: "24"
  interface: external

onos_nic: eth2
tenant_net_info:
  type: vxlan
  range: "1000:1050"
  provider_network: None

public_net_info:
  enable: "True"
  network: ext-net
  type: flat
  segment_id: 50
  subnet: ext-subnet
  provider_network: physnet
  router: router-ext
  enable_dhcp: "False"
  no_gateway: "False"
  external_gw: "192.168.20.1"
  floating_ip_cidr: "192.168.20.0/24"
  floating_ip_start: "192.168.20.105"
  floating_ip_end: "192.168.20.200"
```

#### 3.1.2 策略配置文件

该文件主要是电源管理以及针对各个节点的角色分配，详细介绍见官方[Nodes Configuration (Bare Metal Deployment)](https://opnfv-compass4nfv.readthedocs.io/en/stable-gambia/release/installation/bmdeploy.html)

```shell
opnfv@ubuntu:~/compass4nfv/deploy/conf/hardware_environment/bii-pod1$ cat os-nosdn-nofeature-ha.yml

---
TYPE: baremetal
FLAVOR: cluster
POWER_TOOL: ipmitool   #电源管理工具

ipmiUser: root
ipmiVer: '2.0'

hosts:
  - name: ctl01
    mac: '44:A8:42:1A:49:A5'
    interfaces:
      - eth0: '44:a8:42:14:cd:0d'
    ipmiIp: 192.168.20.203
    ipmiPass: admin
    roles:
      - controller
      - ha
      - ceph-adm
      - ceph-mon

  - name: ctl02
    mac: '44:A8:42:1A:76:2C'
    interfaces:
      - eth0: '44:a8:42:15:1b:e6'
    ipmiIp: 192.168.20.204
    ipmiPass: admin
    roles:
      - controller
      - ha
      - ceph-mon

  - name: ctl03
    mac: '44:A8:42:13:D5:1B'
    interfaces:
      - eth0: '44:a8:42:14:fc:1a'
    ipmiIp: 192.168.20.205
    ipmiPass: admin
    roles:
      - controller
      - ha
      - ceph-mon

  - name: cmp001
    mac: '44:A8:42:1A:70:BE'
    interfaces:
      - eth0: '44:a8:42:14:ee:64'
    ipmiIp: 192.168.20.201
    ipmiPass: admin
    roles:
      - compute
      - ceph-osd

  - name: cmp002
    mac: '44:A8:42:1A:76:26'
    interfaces:
      - eth0: '44:a8:42:14:cb:31'
    ipmiIp: 192.168.20.202
    ipmiPass: admin
    roles:
      - compute
      - ceph-osd
```

#### 3.1.3 其他配置

1) DNS及时区设置

考虑到网络等原因可以自定义DNS和时区，修改配置文件`compass4nfv/deploy/conf/compass.conf`中的DNS和时区，同时在这里我们也可以看到compass的web界面的账号和密码

```shell
export COMPASS_DECK_PORT="5050"

export COMPASS_USER_EMAIL="admin@huawei.com"
export COMPASS_USER_PASSWORD="admin"
```

访问安装节点的5050端口即可打开页面查看部署节点信息

![compass登录界面](https://raw.githubusercontent.com/louielong/blogPic/master/imgVEG8XIG.jpg)

2）docker镜像配置

compass在部署过程中会安装docker并下载相应镜像，可以配置国内源加速镜像下载

修改`compass4nfv/deploy/prepare.sh`中的docker配置部分，增加中科大的docker镜像源或者其他源均可。(采用离线部署时所需的docker镜像都打包下载了，所以是否修改不影响)

```shell
     sudo cat << EOF > /etc/docker/daemon.json
 {
   "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/"],
   "storage-driver": "devicemapper"
 }
 EOF
```

### 3.2 部署脚本配置

部署脚本为`deploy.sh`文件，主要修改的地方节选如下

```shell
# Set OS version for target hosts
# Ubuntu16.04 or CentOS7
export OS_VERSION=xenial     #指定待部署节点的系统版本
# Set ISO image corresponding to your code
export TAR_URL=file:///home/opnfv/download/opnfv-6.2-m.tar.gz   #指定下载的系统安装文件

#export DEPLOY_HARBOR="true"
#export HABOR_VERSION="1.5.0"

# Set hardware deploy jumpserver PXE NIC
# You need to comment out it when virtual deploy.
export INSTALL_NIC=eth0  #指定PXE网卡
# DHA is your dha.yml's path
export DHA=/home/opnfv/compass4nfv/deploy/conf/hardware_environment/bii-pod1/os-nosdn-nofeature-ha.yml  #指定部署策略模板

# NETWORK is your network.yml's path
export NETWORK=/home/opnfv/compass4nfv/deploy/conf/hardware_environment/bii-pod1/network.yml     #指定待部署节点的网络配置

#export OPENSTACK_VERSION=${OPENSTACK_VERSION:-ocata}
export OPENSTACK_VERSION=queens  #指定部署的Openstack版本
```

### 3.3 部署opnfv

本节会分析并记录部署中出现的问题以及排查解决办法。

进入节点的方法，首先进入到`compass-tasks`容器中，然后直接使用ssh登录

```shell
opnfv@ubuntu:~$ docker exec -it compass-tasks bash

[root@compass-tasks /]# ssh root@10.20.0.53
Warning: Permanently added '10.20.0.53' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-87-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Fri Nov 23 17:23:21 2018

root@cmp001:~#

```

在后续的排错中也发现了节点可以使用密码登录，密码记录在`compass4nfv/deploy/adapters/cobbler/kickstarts/default16.seed`中的`root-password`一行，为`user/passwd：root/root`。

**【Note】**

在后续部署中出现错误可以查看对应的ansible任务*task*，首先在`compass4nfv/deploy/adapters/ansible`目录下查看对应的`task`详细执行命令，其次在`compass-tasks`的容器中的`/etc/ansible`目录下查看。

#### 3.3.1 ubuntu安装无法自动跳过NTP检测

我在安装过程中一直被ubuntu安装时的“setting up the clock”卡住，无法自动跳过，只能手动跳过，这个是ubuntu 16.04的一个bug，可能是跟配置网络的其他ntp服务器有关。

![setting_time_check](https://raw.githubusercontent.com/louielong/blogPic/master/img9CJUyM5.png)

根据[2]修改系统安装时的prseed文件，屏蔽掉系统安装时的ntp检查，修改`compass4nfv/deploy/adapters/cobbler/kickstarts`下的`default16.seed`文件中的

> d-i clock-setup/ntp boolean true

改成

> d-i clock-setup/ntp boolean false

**【Note】**

*Fuel*部署工具使用的是Debian系的MAAS来进行无人值守安装系统（免费版可以安装ubuntu，但是安装centos则需要付费），而*Compass4nfv*使用的是RedHat系的*kickstart*和*cobbler*来进行无人值守部署系统，深入阅读可以查看[KICKSTART无人值守安装](http://www.zyops.com/autoinstall-kickstart/)
，[Ubuntu系统批量自动安装](https://www.jianshu.com/p/97ef50f06f05)或者其他相关资料。

#### 3.3.2 本地ubuntu源配置

由于ubuntu的本地源使用率较高，考虑到网络带宽以及资源复用的方面，可以在本地建立ubuntu源，ubuntu源的搭建可以参考我的之前文章[MAAS+ubuntu私有源环境搭建](http://ylong.net.cn/MAAS+ubuntu-local-repo.html)。这里为了加快安装后系统的ubuntu软件安装，使用本地源进行软件安装。

主要修改的如下两个文件的仓库IP地址

1）compass4nfv/deploy/adapters/ansible/roles/config-compute/vars/main.yml

2）compass4nfv/deploy/adapters/ansible/roles/config-controller/vars/main.yml

> LOCAL_REPOSITORY_IP: "192.168.137.222"
>

将IP地址修改为本地镜像仓库地址，Compass在部署过程中会去检测这个IP的连通性，若IP地址不通仍然使用ubuntu官方的镜像源地址，如文件`compass4nfv/deploy/adapters/ansible/roles/config-controller/templates/sources.list.official`所示。

#### ~~3.3.3 ubuntu源更新禁用IPv6设置~~

我的网络是带有IPv6，ubuntu在源更新时会优先走IPv6，但是IPv6网络无法更新(部署执行中报错，但是我登陆节点直接执行`apt update`却能成功，目前原因未知)只能现行禁用apt源更新走IPv6[3].

解决方法1：

增加配置文`件99force-ipv4 `，加入`Acquire::ForceIPv4 "true";`，配置文件放置在`compass4nfv/deploy/adapters/ansible/roles/config-controller/files`

随后在`compass4nfv/deploy/adapters/ansible/roles/config-controller/vars/Ubuntu.yml`的任务中添加拷贝文件的命令

```yaml
- name: apt force IPv4
  copy:
    src: 99force-ipv4
    dst: /etc/apt/apt.conf.d/99force-ipv4
```

解决办法2：

使用alias在apt命令后增加`-o Acquire::ForceIPv4=true`选项，修改`compass4nfv/deploy/adapters/ansible/roles/config-controller/vars/Ubuntu.yml`，在`name: add apt.conf`前添加

```yaml
- name: add apt-get alias
  shell: "echo "alias apt-get='apt-get -o Acquire::ForceIPv4=true'" >> /etc/bash.bashrc"

- name: add apt.conf
  copy:
    src: apt.conf
    dest: /etc/apt/apt.conf
  when: offline_deployment is defined and offline_deployment == "Enable"
```

**【NOTE】**

同时也能注意到`compass/deploy/adapters/ansible/roles/config-compute/files/apt.conf`为了避免apt软件安装中出现`[y/N]`这样的交互，采用以下参数来配置apt安装，相关资料链接[传送门](https://superuser.com/questions/164553/automatically-answer-yes-when-using-apt-get-install)

> APT::Get::Assume-Yes "true";
> APT::Get::force-yes "true";

#### 3.3.4 apt源更新时网络未连通

我在安装过程中出现无法ping通内网仓库IP的以及后续初始apt更新源失败的情况，原因是网卡重启后网络未连通导致，这里适当延长了网卡重启后的时间，修改`compass4nfv/deploy/adapters/ansible/roles/config-controller/tasks/Ubuntu.yml`

```yaml
- name: check apt source
  shell: "sleep 60 && ping -c 2 {{ LOCAL_REPOSITORY_IP }} > /dev/null"
  register: checkresult
  ignore_errors: "true"
```

后续的安装中除了多次出现无法下载*Openstack*官方的*Git*[仓库代码](https://git.openstack.org/openstack)而报错外，并未遇到什么大的问题，而对于这个问题我暂时也没有太好的解决办法，只能进入容器内部尝试查看具体的网络错误原因。

compass4nfv在安装OPNFV采用的是lxc的容器安装的，会在每个节点生成对应服务的lxc容器，可以通过`lxc-attach --name  ctl02_repo_container-5df4d773` 进入到容器进行查看。

```shell
root@ctl02:/openstack/ctl02_repo_container-5df4d773/repo/openstackgit# lxc-ls
ctl02_aodh_container-95ac1216 ctl02_ceilometer_central_container-fe83768b ctl02_ceph-mon_container-6b5f3bdb ctl02_cinder_api_container-edae48f8
ctl02_galera_container-4514e3cb  ctl02_glance_container-c8150d02 ctl02_gnocchi_container-add47e23 ctl02_heat_api_container-2a7289ca
ctl02_horizon_container-c48e853e ctl02_keystone_container-b6189735   ctl02_memcached_container-33866197  ctl02_neutron_server_container-8638a6eb
ctl02_nova_api_container-7221202a ctl02_rabbit_mq_container-e8f31a4e ctl02_repo_container-5df4d773 ctl02_rsyslog_container-97cbc806
ctl02_tacker_container-84a1e0b9 ctl02_utility_container-d08003d6
```

同样的，会将openstack官方的稳定版代码下载到本地的`/openstack/ctl02_repo_container-5df4d773/repo/openstackgit`对应项目目录下，而对应容器内部则是在`/var/www/repo/opensatck`目录下。

```shell
root@ctl02-repo-container-5df4d773:/var/www/repo/openstackgit# ls
aodh dragonflow heat-dashboard magnum-ui networking-sfc neutron-fwaas-dashboard nova octavia-dashboard spice-html5
ceilometer glance horizon networking-bgpvpn neutron neutron-lbaas nova-lxd rally    tacker
cinder gnocchi ironic-ui networking-calico neutron-dynamic-routing neutron-lbaas-dashboard nova-powervm requirements tempest
designate-dashboard heat keystone networking-odl neutron-fwaas neutron-vpnaas novnc sahara-dashboard trove-dashboard
```

#### 3.3.5 ceph安装错误

由于国内网络原因在ceph的安装过程中会由于网络不通或者下载过慢超时等出现ceph安装出错，这里修改ceph的源[4][5]，可以选择的国内源有：

- 阿里镜像源<http://mirrors.aliyun.com/ceph>
- 网易镜像源<http://mirrors.163.com/ceph>
- 中科大镜像源<http://mirrors.ustc.edu.cn/ceph>

这里以阿里云的源为例，各大源只有`rethat`系和`debian`系两种分类，这里的`ubuntu 16.04`使用的是`debian-luminous`的版本，查看`ctl`节点新增的apt源可以确认，后续随着安装使用的ubuntu版本升级会有所改变，需要灵活处理

```shell
root@ubuntu:~# ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@10.20.0.51
...
root@10.20.0.51's password:
...
You have new mail.
Last login: Tue Dec 11 10:02:26 2018 from 10.20.0.1
root@ctl02:~# lxc-attach --name ctl02_ceph-mon_container-2a24ec76
root@ctl02-ceph-mon-container-2a24ec76:~# cat /etc/apt/sources.list.d/download_ceph_com_debian_luminous.list
#deb http://download.ceph.com/debian-luminous xenial main
deb http://mirrors.ustc.edu.cn/ceph/debian-luminous/ xenial main
```

安装的源码修改如下:

由于这部分操作是在容器`compass-tasks`中操作的，可以在部署源码中添加修改，也可以在部署过程中直接在容器中修改

方法一：

1）新增文件

   路径为`compass4nfv/util/docker-compose/roles/compass/files/debian_community_repository.yml`

   ```shell
opnfv@ubuntu:~/compass4nfv/util/docker-compose/roles/compass/files$ cat debian_community_repository.yml
---
- name: configure debian ceph community repository stable key
  apt_key:
    url: http://mirrors.aliyun.com/ceph/keys/release.asc
    state: present

- name: configure debian ceph stable community repository
  apt_repository:
    repo: "deb http://mirrors.ustc.edu.cn/ceph/debian-luminous/ xenial main"
    state: present
    update_cache: yes
  changed_when: false
   ```

2）增加ansible的task

修改文件`compass4nfv/deploy/deploy_host.sh`，在`export AYNC_TIMEOUT=20`后增加以下内容

```shell
docker cp \
$COMPASS_DIR/util/docker-compose/roles/compass/files/debian_community_repository.yml \
compass-tasks:/etc/ansible/roles/ceph-ansible/roles/ceph-common/tasks/installs/debian_community_repository.yml
```

方法二：

由于部署ceph在靠后的阶段，因此可以在容器`compass-tasks`起来后直接修改

```shell
root@ubuntu:/home/opnfv/compass4nfv/deploy# docker exec -it compass-tasks bash
[root@compass-tasks /]# cat /etc/ansible/roles/ceph-ansible/roles/ceph-common/tasks/installs/debian_community_repository.yml

---
- name: configure debian ceph community repository stable key
  apt_key:
    url: "http://mirrors.aliyun.com/ceph/keys/release.asc"
    state: present

- name: configure debian ceph stable community repository
  apt_repository:
    repo: "deb http://mirrors.aliyun.com/ceph/debian-luminous/ xenial main"
    state: present
    update_cache: yes
  changed_when: false
```

#### 3.3.6 无法下载HaTop

依然是网络原因导致节点无法下载Hatop安装文件

```shell
 fatal: [ctl02 -> localhost]: FAILED! => {"changed": false, "failed": true, "msg": "Failed to connect to storage.googleapis.com at port 443: [      Errno 99] Cannot assign requested address"}
```

可以预选下载[Hatop](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/hatop/hatop-0.7.7.tar.gz)放在本地的一个文件服务器上，直接修改`compass-tasks`中`/etc/ansible/roles/haproxy_server/defaults/main.yml`的`haproxy_hatop_download_url`为自定义的文件服务器。

```shell
root@ubuntu:~# docker exec -it compass-tasks bash
[root@compass-tasks /]# vim /etc/ansible/roles/haproxy_server/defaults/main.yml
...
haproxy_hatop_download_url: "https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/hatop/hatop-0.7.7.tar.gz"
...
```

或者跟`3.3.5`节一样在文件`compass4nfv/deploy/deploy_host.sh`，在`export AYNC_TIMEOUT=20`后增加以下内容

```shell
docker exec compass-tasks bash -c \
"sed -i 's/https:\/\/stor[^ ]*gz/http:\/\/192.168.6.231\/download\/hatop-0.7.7.tar.gz/' /etc/ansible/roles/haproxy_server/defaults/main.yml"
```

#### 3.3.7 部署超时

compass在部署时设定了超时时间为300分钟即5个小时，但是由于由内网络下载等原因会导致部署超时，出现如下错误

> Traceback (most recent call last):
> File "/home/opnfv/compass4nfv/deploy/client.py", line 1136, in <module>
>  main()
> File "/home/opnfv/compass4nfv/deploy/client.py", line 1131, in main
>  deploy()
> File "/home/opnfv/compass4nfv/deploy/client.py", line 1086, in deploy
>  client.get_installing_progress(cluster_id, ansible_print)
> File "/home/opnfv/compass4nfv/deploy/client.py", line 1029, in get_installing_progress
>  _get_installing_progress()
> File "/home/opnfv/compass4nfv/deploy/client.py", line 1026, in _get_installing_progress
>  raise RuntimeError("installation timeout")
> RuntimeError: installation timeout

我在部署时其时长都会超过300分钟，因此这里加大部署超时时间，修改`compass4nfv/deploy/conf/baremetal.conf`

```shell
export DEPLOYMENT_TIMEOUT="1000"
```

#### 3.3.8 openstack官方仓库clone失败

compass部署过程中会从openstack的[官方仓库](https://git.openstack.org/cgit)下载稳定分支的代码用于部署，同样是国内网络的原因会出现clone失败的情况，如下所示当计数变为1时仍未下载完成就会部署失败。

> 2018-12-20 06:49:01,341 p=33580 u=root |  FAILED - RETRYING: Wait for git clones to complete (1 retries left).
> 2018-12-20 06:49:06,574 p=33580 u=root |  FAILED - RETRYING: Wait for git clones to complete (1 retries left).
> 2018-12-20 06:49:11,796 p=33580 u=root |  FAILED - RETRYING: Wait for git clones to complete (1 retries left).

上述错误并不是每次都会出现，为了简单解决我选择将下载成功的代码打包备份到Master部署节点，随后在安装过程中拷贝至控制节点解压，~~需要等到控制节点上的`ctl02_repo_container-xxxx`容器（由于ctl02是默认的主控制节点，因此只用传给ctl02即可）创建成功后才行~~，这里将拷贝和解压直接集成到部署代码中。

压缩代码，将`repo`目录下的其他文件夹删除只留下openstackgit，随后将整个repo目录打包为`repo.tar.gz`压缩代码(800M起)。

```shell
root@ctl02:/openstack/ctl02_repo_container-10dffa61/repo# ls
links  openstackgit  os-releases  pkg-cache  pools  repo_prepost_cmd.sh  venvs
root@ctl02:/openstack/ctl02_repo_container-10dffa61# tar czf repo.tar.gz repo
root@ctl02:/openstack/ctl02_repo_container-10dffa61# ls -ahl repo.tar.gz
-rw-r--r-- 1 root root 831M Jan 16 14:56 /home/opnfv/repo.tar.gz
```

将`repo.tar.gz`上传至master节点待用，修改部署脚本，在部署中首先将`repo.tar.gz`拷贝到`compass-tasks`容器中，随后游部署脚本将其拷贝到ctl02_repo_container-xxxxx容器中解压到`/var/www`目录下

1)首先创建ansible任务，

```shell
root@ubuntu:/home/opnfv/compass4nfv# cat deploy/adapters/ansible/customer/git_repo_pre.yml

- name: scp repo.tar.gz
  copy:
    src: /opt/ansible_plugins/repo.tar.gz
    dest: /root/repo.tar.gz
    mode: 0644
  ignore_errors: yes

- name: ungzip repo.tar.gz
  shell:
    tar xf /root/repo.tar.gz -C /var/www/ --remove-files
  ignore_errors: yes
```

2)修改部署脚本

在文件`compass4nfv/deploy/deploy_host.sh`，在`export AYNC_TIMEOUT=20`后增加以下内容

```shell
# git repo
cp -a /home/opnfv/repo.tar.gz /home/opnfv/compass4nfv/work/deploy/docker/ansible_plugins
docker cp \
$COMPASS_DIR/deploy/adapters/ansible/customer/git_repo_pre.yml \
compass-tasks:/etc/ansible/roles/repo_build/tasks/git_repo_pre.yml
docker exec compass-tasks bash -c \
"sed -i '/Create package directories/i\- include: git_repo_pre.yml' /etc/ansible/roles/repo_build/tasks/repo_build_prepare.yml"
```

#### 3.3.9 修改节点external网络的DNS

打开文件`deploy/adapters/ansible/roles/config-compute/templates/ifcfg-br-external`，修改dns配置即可，比如使用国内阿里云的dns，同理修改控制节点文件

```shell
DNS1=223.5.5.5
DNS2=8.8.8.8
```

#### 3.3.10 lxc镜像下载失败

> 2018-12-21 02:46:31,836 p=4989 u=root |  FAILED - RETRYING: Ensure image has been pre-staged (1 retries left).
> 2018-12-21 02:46:36,923 p=4989 u=root |  FAILED - RETRYING: Ensure image has been pre-staged (1 retries left).
> 2018-12-21 02:46:37,015 p=4989 u=root |  FAILED - RETRYING: Ensure image has been pre-staged (1 retries left).
> 2018-12-21 02:46:37,017 p=4989 u=root |  FAILED - RETRYING: Ensure image has been pre-staged (1 retries left).

又是一个网络引起的部署错误，每次部署都会去下载最新的`ubuntu lxc`镜像，由于网络原因，有时能够下载成功，有时下载失败，如图所示的小水管，虽然只有70M，但是也会出现无法在限定时间300s内下载完成

![lxc镜像下载](https://raw.githubusercontent.com/louielong/blogPic/master/imgHk9mbqz.png)

这里使用[清华源的lxc镜像](https://mirrors.tuna.tsinghua.edu.cn/lxc-images/images/ubuntu/xenial/amd64/default/)，其会自动同步LXC官方镜像，并保持相同的链接格式(对链接格式感兴趣的可以看3.3.10.1小节)，镜像地址在`compass-tasks`镜像的`/etc/ansible/roles/lxc_hosts/defaults/main.yml`第160行。

由于部署工具会依据官方的镜像列表筛选出符合要求的最新镜像，

```shell
aria2c --max-connection-per-server=4 --allow-overwrite=true --dir=/tmp --out=index-system --check-certificate=true  https://mirrors.tuna.tsinghua.edu.cn/lxc-images/meta/1.0/index-system
```

这里在容器启动后修改，同样是修改文件`compass4nfv/deploy/deploy_host.sh`，在`export AYNC_TIMEOUT=20`后增加以下内容。

```shell
docker exec compass-tasks bash -c \
"sed -i 's/us.images.linuxcontainers.org/mirrors.tuna.tsinghua.edu.cn\/lxc-images/' /etc/ansible/roles/lxc_hosts/defaults/main.yml"
```

修改后的测试，如图所示，直接从80K/s飙到了9.4M/s

![清华源LXC镜像下载](https://raw.githubusercontent.com/louielong/blogPic/master/imgqgny2Yv.png)

```shell
aria2c --max-connection-per-server=4 --allow-overwrite=true --dir=/tmp/test --out=rootfs.tar.xz --check-certificate=true https://mirrors.tuna.tsinghua.edu.cn/lxc-images/images/ubuntu/xenial/amd64/default/20190106_07:43/rootfs.tar.xz
```

##### 3.3.10.1 LXC官方镜像下载链接格式说明

由于LXC官方会更新最新的镜像，因此下载的链接会稍有不同，LXC镜像链接：[Linux Containers - Image server](https://us.images.linuxcontainers.org/)

![LXC ubuntu镜像列表](https://raw.githubusercontent.com/louielong/blogPic/master/imgkNMLCVd.png)

以`ubuntu;xenial;amd64;default;20190106_07:43;`为例，其下载链接组成为`ubuntu/xenial/amd64/default/20190106_07:43`，完整的下载地址为：

`https://us.images.linuxcontainers.org/images/ubuntu/xenial/amd64/default/20190106_07:43/rootfs.tar.xz`

#### 3.3.11 lxc网桥断开连接

在安装的后期阶段由于external网络从linux bridge切换到ovs会出现lxc容器无法连接网络而导致安装失败，这里采用lxc网络自检工具定时检测网络状态，在lxc容器创建完成后，分别在**三个控制节点**添加定时任务

```shell
root@ctl02:~# crontab -e

*/3 * * * * /bin/echo -n "$(date +"\%F \%T") " >> /tmp/lxc-veth-check;/usr/local/bin/lxc-veth-check >> /tmp/lxc-veth-check
```

亦可以修改部署脚本自动添加lxc veth网卡检查

-  1）创建文件`compass4nfv/deploy/adapters/ansible/custome/lxc_veth_check.yml`，写入如下内容

```yaml
- name: Add lxc veth check cron job
  cron:
    name: check lxc veth
    minute: "*/3"
    job: '*/3 * * * * /bin/echo -n "$(date +"\%F \%T") " >> /tmp/lxc-veth-check;/usr/local/bin/lxc-veth-check >> /tmp/lxc-veth-check'
    state: present
```

- 2）修改部署脚本拷贝`lxc_veth_check.yml`并增加ansible部署

修改`compass4nfv/deploy/deploy_host.sh`，在`export AYNC_TIMEOUT=20`后增加以下内容

```shell
# lxc veth check
docker cp \
$COMPASS_DIR/deploy/adapters/ansible/custome/lxc_veth_check.yml \
compass-tasks:/etc/ansible/roles/lxc_container_create/tasks/lxc_veth_check.yml
docker exec compass-tasks bash -c \
"echo '- include: lxc_veth_check.yml' >> /etc/ansible/roles/lxc_container_create/tasks/main.yml"
```







## 四 使用

在费劲千辛万苦终于安装好OPNFV后下面要面对的就是如何使用，可以参考compass的使用手册[6]

### 4.1 Dashboard登录

根据之前`network.yml`中配置的`public_vip`地址进行访问即可，账号密码在控制节点的`openrc`文件中，选择一个控制节点即可看到，如果没有可以去容器`ctl02_utility_container-xxxx`中查看

```shell
root@ctl02:~# ls
openrc
root@ctl02:~# cat openrc
# Ansible managed
export LC_ALL=C

# COMMON CINDER ENVS
export CINDER_ENDPOINT_TYPE=internalURL

# COMMON NOVA ENVS
export NOVA_ENDPOINT_TYPE=internalURL

# COMMON OPENSTACK ENVS
export OS_ENDPOINT_TYPE=internalURL
export OS_INTERFACE=internalURL
export OS_USERNAME=admin
export OS_PASSWORD='292f46714fed1f472e885efbbbb8b8f0a459ccea'
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://10.20.0.100:5000/v3
export OS_NO_CACHE=1
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_REGION_NAME=RegionOne

# For openstackclient
export OS_IDENTITY_API_VERSION=3
export OS_AUTH_VERSION=3

root@ctl02:~# lxc-ls
ctl02_aodh_container-6124c4b3               ctl02_ceilometer_central_container-27019a4c ctl02_ceph-mon_container-85c1ae54           ctl02_cinder_api_container-be31235f
ctl02_galera_container-88babcd8             ctl02_glance_container-babbb22f             ctl02_gnocchi_container-61f866d0            ctl02_heat_api_container-56ac3d29
ctl02_horizon_container-68362989            ctl02_keystone_container-29eec14f           ctl02_memcached_container-9b21550c          ctl02_neutron_server_container-0e1ade82
ctl02_nova_api_container-60be045b           ctl02_rabbit_mq_container-301a6ec0          ctl02_repo_container-0b096846               ctl02_rsyslog_container-aa15bf1b
ctl02_tacker_container-2d6471ad             ctl02_utility_container-bd689ebc
root@ctl02:~# lxc-attach --name  ctl02_utility_container-bd689ebc
root@ctl02-utility-container-bd689ebc:~# ls
openrc

```







**【未完待续】**



## 【参考链接】

1）[Compass 介绍](https://opnfv-compass4nfv.readthedocs.io/en/stable-gambia/release/installation/index.html#compass4nfv-installation)

2）[ubuntu 时钟设置bug](https://bugs.launchpad.net/ubuntu/+source/debian-installer/+bug/1558166)

3）[ubuntu apt禁用IPv6](http://forum.ubuntu.org.cn/viewtopic.php?t=474276)

4）[使用国内源部署ceph](http://xiaoquqi.github.io/blog/2016/06/19/deploy-ceph-using-china-mirror/)

5）[ceph国内源](http://blog.51cto.com/opencloud/1948400)

6）[compass部署opnfv使用手册](https://wiki.opnfv.org/display/compass4nfv/Containerized+Compass)

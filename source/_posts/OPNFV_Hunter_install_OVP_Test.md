---
title: OPNFV Hunter Fuel安装以及OVP测试使用
date: 2019-05-27 13:49:55
tags:
  - OPNFV
  - OVP
  - Dovetail
  - Fuel
keywords:
  - OPNFV
  - Dovetail
categories:
  - OPNFV
description:
  - OPNFV Hunter版本安装以及OVP测试使用
summary_img:
  - https://raw.githubusercontent.com/louielong/blogPic/master/imgOPNFV_Hunter_Assets_v1_WebBanner.png
---

## 一、前言

OPNFV与5月发布了第八个版本Hunter，随即升级了实验室的环境，同时也测试一下即将发布的OVP 第三个版本。官方社区的安装工具也减少到只剩下**MCP (Fuel)**和**TripleO**两个了，Fuel的安装方式没有太多的改变，只是引入了**Docker**将原来的*fuel*和*MAAS*两个安装虚拟机换成了Docker容器。实际在部署过程中发现与先前的[OPNFV Euphrates部署 02](opnfv_Euphrates_install-02.html)和[OPNFV Euphrates部署 03](opnfv_Euphrates_install-02.html)没有区别，在配置好**网络**、**PDF**以及**IDF**后可以很顺利的安装。同时看到Fuel部署的文档写的更加详细了，参见：

- [OPNFV Hunter Fuel安装指导](https://docs.opnfv.org/en/stable-fraser/submodules/fuel/docs/release/installation/installation.instruction.html)
- [OPNFV Fuel 用户手册](https://opnfv-fuel.readthedocs.io/en/stable-hunter/release/userguide/userguide.html)

OVP也将发布第三个版本OVP-2019.08增强测试内容，因此这里一并整理一下使用与问题排查办法。



## 二、Hunter 部署

Fuel的Hunter部署和Euphrates以及Gambia没有太大的差别，完全可以参考先前的[OPNFV Euphrates部署 02](opnfv_Euphrates_install-02.html)和[OPNFV Euphrates部署 03](opnfv_Euphrates_install-02.html)，这里主要介绍一下加快部署的办法。

## 2.1 MAAS local mirros

部署过程中遇到的最大的问题就是MAAS每次都会去重新获取最新的ubuntu镜像，常常因为镜像下载缓慢导致部署超时，这里可以参看[MAAS本地源设置](./MAAS+ubuntu-local-repo.html)，配置本地MAAS源以加快部署

修改`mcp/reclass/classes/cluster/all-mcp-arch-common/infra/maas.yml.j2`

```yaml
      boot_sources:
        resources_mirror:
          #url: http://images.maas.io/ephemeral-v3/daily
          url: http://<IMG mirror IP>/maas/images/ephemeral-v3/daily
          keyring_file: /usr/share/keyrings/ubuntu-cloudimage-keyring.gpg
      boot_sources_selections:
        xenial:
          #url: "http://images.maas.io/ephemeral-v3/daily"
          url: "http://<IMG mirror IP>/maas/images/ephemeral-v3/daily"
```



## 2.2  dashboard不可用

本次安装在解决完MAAS本地源后安装十分顺利，但是安装完后无法使用dashboard，给社区提了一个jira [Fuel-408](https://jira.opnfv.org/browse/FUEL-408)看后续修复吧，尝试在本地解决了一些问题，使得访问**prx01/prx02**的的8078端口可以看到正常的页面，但是使用密码无法登录。



## 【Note】

这里记录一下在无dashboard的情况下配置并测试openstack环境

### 1)上传镜像

```shell
openstack image create cirros --file cirros-0.3.5.qcow2 --container-format bare --public
```

### 2)创建external网络

```shell
openstack subnet create --network floating_net --allocation-pool start=192.168.20.100,end=192.168.20.200  --dns-nameserver 10.2.2.20 --gateway 192.168.20.1 --subnet-range 192.168.20.0/1 floating_subnet
```

### 3)创建内部虚拟网络

```shell
openstack network create test
openstack subnet create worker_subnet --network worker_network --subnet-range 10.20.30.0/24
```

### 4)创建路由

```shell
openstack floating ip create floating_net
openstack router create router
openstack router set router --external-gateway floating_net
openstack router add subnet router test_subnet
```

### 5)安全组配置

```shell
openstack security group rule create <default group id> --protocol tcp --dst-port 22:22 --remote-ip 0.0.0.0/0
openstack security group rule create --protocol icmp <default group id>
```

### 6)创建flavor

```shell
openstack flavor create m1.tiny --ram 64 --disk 0 --vcpus 1 --public
nova flavor-key <ID> set hw:mem_page_size=large  # 由于使用的dpdk需要配置大页内存
```

### 7)启动虚拟机

```shell
nova boot --flavor m1.tiny --image cirros --nic net-id=<floating_net id> test
```

### 8)分配浮动IP

```shell
openstack floating ip create floating_net
openstack floating ip set --port <port> <flaoting ip id>
```



## 三、OVP测试使用

OVP测试使用参考：[OVP用户手册](https://opnfv-dovetail.readthedocs.io/en/latest/testing/user/userguide/testing_guide.html)

选择一台机器，安装配置好docker，并确保机器可以访问OPNFV环境的keystone API

### 3.1 测试环境配置

#### 1）设置dovetial home目录

```shell
$ mkdir -p ${HOME}/dovetail
$ export DOVETAIL_HOME=${HOME}/dovetail
```

#### 2)配置文件设置

```shell
$ mkdir -p ${DOVETAIL_HOME}/pre_config
$ mkdir -p ${DOVETAIL_HOME}/images
```

填写配置文件内容

该文件的内容可以从控制节点**ctl0x**的`/root/keystonercv3`获取。

```shell
$ cat ${DOVETAIL_HOME}/pre_config/env_config.sh

# Project-level authentication scope (name or ID), recommend admin project.
export OS_PROJECT_NAME=admin

# For identity v2, it uses OS_TENANT_NAME rather than OS_PROJECT_NAME.
export OS_TENANT_NAME=admin

# Authentication username, belongs to the project above, recommend admin user.
export OS_USERNAME=admin

# Authentication password. Use your own password
export OS_PASSWORD=opnfv_secret

# Authentication URL, one of the endpoints of keystone service. If this is v3 version,
# there need some extra variables as follows.
export OS_AUTH_URL=http://xxxxxxxxxxxx:35357/v3

export OS_IDENTITY_API_VERSION=3

# Domain name or ID containing the user above.
# Command to check the domain: openstack user show <OS_USERNAME>
export OS_USER_DOMAIN_NAME=Default

# Domain name or ID containing the project above.
# Command to check the domain: openstack project show <OS_PROJECT_NAME>
export OS_PROJECT_DOMAIN_NAME=Default

# Special environment parameters for https.
# If using https + cacert, the path of cacert file should be provided.
# The cacert file should be put at $DOVETAIL_HOME/pre_config.
export OS_CACERT=/path_to/pre_config/os_cacert
export OS_INSECURE=false

export OS_REGION_NAME=RegionOne
export OS_INTERFACE=internal
export OS_ENDPOINT_TYPE="internal"
export INSTALLER_TYPE=fuel
export EXTERNAL_NETWORK="floating_net"
#export VOLUME_DEVICE_NAME=vdc

# Set an existing role used to create project and user for vping test cases.
# Otherwise, it will create a role 'Member' to do that.
export NEW_USER_ROLE=xxx
```

配置`Tempest`所需的配置文件，编辑文件`$DOVETAIL_HOME/pre_config/tempest_conf.yaml`

```shell
compute:
  # The minimum number of compute nodes expected.
  # This should be no less than 2 and no larger than the compute nodes the SUT actually has.
  min_compute_nodes: 2

  # Expected device name when a volume is attached to an instance.
  volume_device_name: vdb
```

节点描述信息，dovetail在测试时会获取节点信息，同时在HA测试部分会重启节点也需要用到节点信息

```shell
nodes:
-
    # This can not be changed and must be node0.
    name: node0
    role: Jumpserver
    ip: xx.xx.xx.xx
    user: root
    password: root

-
    # This can not be changed and must be node1.
    name: node1
    # This must be controller.
    role: Controller
    # This is the instance IP of a controller node, which is the haproxy primary node
    ip: xx.xx.xx.xx
    # User name of the user of this node. This user **must** have sudo privileges.
    user: root
    key_filename: /path_to/pre_config/id_rsa

process_info:
-
    # The default attack process of yardstick.ha.rabbitmq is 'rabbitmq-server'.
    # Here can be reset to 'rabbitmq'.
    testcase_name: yardstick.ha.rabbitmq
    attack_process: rabbitmq

-
    # The default attack host for all HA test cases is 'node1'.
    # Here can be reset to any other node given in the section 'nodes'.
    testcase_name: yardstick.ha.glance_api
    attack_host: node2
```

*process_info*表明在测试相应HA用例时yardstick重启哪些**进程**或**节点**

这里给出相应测试用例攻击的进程名

| Test Case Name                | Attack Process Name |
| ----------------------------- | ------------------- |
| yardstick.ha.cinder_api       | cinder-api          |
| yardstick.ha.database         | mysql               |
| yardstick.ha.glance_api       | glance-api          |
| yardstick.ha.haproxy          | haproxy             |
| yardstick.ha.keystone         | keystone            |
| yardstick.ha.neutron_l3_agent | neutron-l3-agent    |
| yardstick.ha.neutron_server   | neutron-server      |
| yardstick.ha.nova_api         | nova-api            |
| yardstick.ha.rabbitmq         | rabbitmq-server     |



### 3.2 镜像下载

这里仅以OVP-2.2为例

```shell
$ wget -nc http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img -P ${DOVETAIL_HOME}/images
$ wget -nc https://cloud-images.ubuntu.com/releases/14.04/release/ubuntu-14.04-server-cloudimg-amd64-disk1.img -P ${DOVETAIL_HOME}/images
$ wget -nc https://cloud-images.ubuntu.com/releases/16.04/release/ubuntu-16.04-server-cloudimg-amd64-disk1.img -P ${DOVETAIL_HOME}/images
$ wget -nc http://repository.cloudifysource.org/cloudify/4.0.1/sp-release/cloudify-manager-premium-4.0.1.qcow2 -P ${DOVETAIL_HOME}/images  

$ sudo docker pull opnfv/dovetail:ovp-2.2.0
ovp-2.2.0: Pulling from opnfv/dovetail
....
Digest: sha256:7449601108ebc5c40f76a5cd9065ca5e18053be643a0eeac778f537719336c29
Status: Downloaded newer image for opnfv/dovetail:ovp-2.2.0
```

### 3.3 启动dovetail容器

```shell
$ sudo docker run --privileged=true -it \
          -e DOVETAIL_HOME=$DOVETAIL_HOME \
          -v $DOVETAIL_HOME:$DOVETAIL_HOME \
          -v /var/run/docker.sock:/var/run/docker.sock \
          --name dovetail \
          opnfv/dovetail:<tag> /bin/bash
```

其中：

- `-e`指定DOVETAIL_HOME变量
- `-v` 映射DOVETAIL_HOME目录

### 3.3 启动测试

进入容器查看测试用例

```shell
$ docker exec -it dovetail bash
root@f02275952f91:~# dovetail list
.....
```

执行测试用例

简单测试

```shell
$ dovetail run --offline --debug --testcase  functest.vping.userdata --deploy-scenario os-nosdn-ovs-ha --report
```

其中：

- **offline**指明为离线测试
- **debug**打开debug选项
- **testcase**指明单项测试用例名
- **deploy-scenario**指明所使用的部署策略，主要影响在于ovs策略中使用了DPDK则会在flavor中附加大页内存配置
- **report**指明生成测试报告

完整测试

OVP认证测试内容分为`mandatory`和`optional`，可以单独执行

```shell
$ dovetail run --mandatory --report
```

### 3.4 测试结果

执行完测试后在`$DOVETAIL_HOME/results`可以获得完整的测试记录，执行不通过的测试用例也可在相应的目录获得日志文件

- Log file: dovetail.log
  - Review the dovetail.log to see if all important information has been captured - in default mode without DEBUG.
  - Review the results.json to see all results data including criteria for PASS or FAIL.
- Tempest and security test cases
  - Can see the log details in `tempest_logs/functest.tempest.XXX.html` and `security_logs/functest.security.XXX.html` respectively, which has the passed, skipped and failed test cases results.
  - This kind of files need to be opened with a web browser.
  - The skipped test cases are accompanied with the reason tag for the users to see why these test cases skipped.
  - The failed test cases have rich debug information for the users to see why these test cases failed.
- Vping test cases
  - Its log is stored in `vping_logs/functest.vping.XXX.log`.
- HA test cases
  - Its log is stored in `ha_logs/yardstick.ha.XXX.log`.
- Stress test cases
  - Its log is stored in `stress_logs/bottlenecks.stress.XXX.log`.
- Snaps test cases
  - Its log is stored in `snaps_logs/functest.snaps.smoke.log`.
- VNF test cases
  - Its log is stored in `vnf_logs/functest.vnf.XXX.log`.

### 3.5 测试失败排查

#### 1)无法获取节点信息

节点信息获取失败并不影响测试执行，可以在dovetail容器中执行以下命令查找失败原因

```shell
ansible all -m setup -i XX/results/inventory.ini --tree XX/results/sut_hardware_info
```

#### 2)dpdk错误相关

我的测试环境是`os-nosdn-ovs-ha`使用了DPDK网卡，有时由于变量传递的原因导致测试过程中，虚机的flavor没有设置`'hw:mem_page_size':'large'`，因此导致虚拟创建后无法连接而报错，可以在测试过程中查看`openstack flavor list --debug`查看是否是此问题，如果是此类问题只能报告官方修复了。

#### 3) 其他测试用例失败

通过dovetail的不清除选项保留测试过程中启动的容器，然后一步一步排查

```shell
$ dovetail run --offline --debug --testcase xxx -n
```

随后进入调用的容器，执行dovetail测试过程中调用的命令，如

```shell
container.Container - DEBUG - Executing command: 'sudo docker run -id --privileged=true  -e INSTALLER_TYPE=unknown -e DEPLOY_SCENARIO=os-nosdn-ovs-ha -e NODE_NAME=unknown -e TEST_DB_URL=file:///home/opnfv/functest/results/functest_results.txt -e CI_DEBUG=true -e BUILD_TAG=daily-master-7c89eac6-829e-11e9-85e8-0242ac140002-functest.tempest.vm_lifecycle       -v /nfs/NFV_TEST/dovetail/data/pre_config/env_config.sh:/home/opnfv/functest/conf/env_file   -v /nfs/NFV_TEST/dovetail/data/pre_config/os_cacert:/nfs/NFV_TEST/dovetail/data/pre_config/os_cacert   -v /nfs/NFV_TEST/dovetail/data:/home/opnfv/userconfig   -v /nfs/NFV_TEST/dovetail/data/results:/home/opnfv/functest/results  -v /nfs/NFV_TEST/dovetail/data/images:/home/opnfv/functest/images opnfv/functest-smoke:opnfv-7.1.0 /bin/bash'
...........
container.Container - DEBUG - Executing command: 'sudo docker exec 75cd8d85a13a /bin/bash -c "run_tests -t tempest_custom -r"'
```

可以进入到`75cd8d85a13a`容器中执行`run_tests -t tempest_custom -r`进一步排查，

对于functest，其代码目录位于容器中的`/usr/lib/python2.7/site-packages/functest`，可以修改`/usr/lib/python2.7/site-packages/functest/ci/logging.ini`中的配置来打开相应组件的日志记录，进行更深层次的排查。



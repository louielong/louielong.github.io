---
title: OPNFV Euphrates 安装（三）
date: 2018-03-06 17:50:43
tags:
  - OPNFV
  - Euphrates
  - dpdk
keywords:
  - dpdk
categories:
  - OPNFV
description:
  - OPNFV Euphrates DPDK scenario deploy
summary_img:
  - https://raw.githubusercontent.com/louielong/blogPic/master/imgmxSHEI5.png
top:
---

## 1 <span id="jump0">前言</span>

近期在研究NFV的网络性能测试，考虑到NFV的网络性能的转发瓶颈，现在的商用NFV产品都会使用诸如DPDK、SR_IOV等网络加速技术，相应的也就需要对应的硬件支持。在尝试部署后本文总结一下使用OPNFV的E版本部署DPDK场景的过程，部署过程中需要修改配置文件以匹配硬件。

## 2 配置文件修改

### 2.1 部署节点配置修改

部署节点的配置文件修改主要是增加DPDK网卡的PCI地址和MAC地址，原配置文件参看[Euphrates部署(二)](http://ylong.net.cn/opnfv_Euphrates_install(02).html)的2.1节。

#### 2.1.1 idf-pod1.yaml修改

在network字段下增加dpdk网卡名，总线地址，接口参数三项

```yaml
    network:
      node:
        # Ordered-list, index should be in sync with node index in PDF
        - interfaces: &interfaces
            # Ordered-list, index should be in sync with interface index in PDF
            - 'eno1'
            - 'eno2'
            - 'eno3'
            - 'eno4'
            - 'enp66s0f0'
            - 'enp66s0f1'
          busaddr: &busaddr
            # Bus-info reported by `ethtool -i ethX`
            - '0000:01:00.0'
            - '0000:01:00.1'
            - '0000:02:00.0'
            - '0000:02:00.1'
            - '0000:42:00.0'
            - '0000:42:00.1'
        - interfaces: *interfaces
          busaddr: *busaddr
        - interfaces: *interfaces
          busaddr: *busaddr
        - interfaces: *interfaces
          busaddr: *busaddr
        - interfaces: *interfaces
          busaddr: *busaddr
        - interfaces: *interfaces
          busaddr: *busaddr
        - interfaces: *interfaces
          busaddr: *busaddr
```

#### 2.1.2 pod1.yaml修改

修改private网络接口为4，接口数按照idf-pod1.yaml的`busaddr`字段下网卡总线地址顺序确定，从0开始计数。

```yaml
  private:
    interface: 4
    vlan: 102
    network: 192.168.102.0
    mask: 24
```

在compute节点上增加DPDK网卡的MAC地址，以及添加`DPDK`特性字段，**Fuel在部署过程中并不是按照节点名称来确定节点类型的，而是按照节点顺序来配置，前三个节点为控制节点，后两个节点为计算节点**。

```yaml
  - name: compute1
    node: *nodeparas
    disks: *disks_A
    remote_management:
      <<: *remote_params
      address: 192.168.20.201
      mac_address: "44:A8:42:1A:70:BE"
    interfaces:                           # physical interface list
      - mac_address: "44:a8:42:14:ee:64"
        speed: 1gb
        features: ''
      - mac_address: "44:a8:42:14:ee:65"
        speed: 1gb
        features: ''
      - mac_address: "44:a8:42:14:ee:66"
        speed: 1gb
        features: ''
      - mac_address: "44:a8:42:14:ee:67"
        speed: 1gb
        features: ''
      - mac_address: "00:1b:21:89:5e:f2"
        speed: 10gb
        feature: 'dpdk'
      - mac_address: "00:1b:21:89:5e:f3"
        speed: 10gb
        feature: 'dpdk'
    fixed_ips:
      admin: 10.20.0.14
      mgmt: 192.168.101.14
      public: 192.168.20.14
```

### 2.2 openstack部署相关配置文件修改

OPNFV的Fuel部署中目前只有一种策略默认支持DPDK，即`os-nosdn-ovs-ha`，打开`mcp/config/scenario/baremetal/os-nosdn-ovs-ha.yaml`文件开一看DPDK字样。

```yaml
---
cluster:
  domain: baremetal-mcp-ocata-ovs-dpdk-ha.local
  states:
    - maas
    - baremetal_init
    - virtual_control_plane
    - dpdk
    - openstack_ha
    - networks
    - networking_gw
virtual:
  nodes:
    - cfg01
    - mas01
  cfg01:
    vcpus: 4
    ram: 6144
  mas01:
    vcpus: 4
    ram: 6144
```

在部署DPDK场景前需要修改配置文件`mcp/reclass/classes/cluster/baremetal-mcp-ocata-ovs-dpdk-ha/openstack/init.yml`[1],配置文件的内容如下：

```yaml
---
classes:
  - cluster.baremetal-mcp-ocata-common-ha.openstack_init
parameters:
  _param:
    neutron_tenant_network_types: "flat,vlan"
    neutron_tenant_vlan_range: "1000:1030"
    nova_cpu_pinning: "1-3,4,6"
    compute_hugepages_size: 2M
    compute_hugepages_count: 8192
    compute_hugepages_mount: /mnt/hugepages_2M
    compute_kernel_isolcpu: 1,2,3,4,5,6,7
    compute_dpdk_driver: uio
    compute_ovs_pmd_cpu_mask: "0x80"
    compute_ovs_dpdk_socket_mem: "2048,2048"
    compute_ovs_dpdk_lcore_mask: "0x20"
    compute_ovs_memory_channels: "2"
  linux:
    system:
      repo:
        uca:
          source: "deb http://ubuntu-cloud.archive.canonical.com/ubuntu xenial-updates/ocata main"
          architectures: amd64
          key_id: EC4926EA
          key_server: keyserver.ubuntu.com
      kernel:
        sysctl:
          net.ipv4.tcp_congestion_control: yeah
          net.ipv4.tcp_slow_start_after_idle: 0
          net.ipv4.tcp_fin_timeout: 30
```

配置文件的参数说明如下：

- neutron_tenant_network_types：表明openstack中将要使用的网络类型，vlan是指创建的虚拟机之间通信的网络类型；
- neutron_tenant_vlan_range：openstack创建的vlan网络vlan范围，需要在物理交换机上进行配置运行该段的vlan数据包通过，否则创建的虚拟机无法进行通信；
- nova_cpu_pinning：计算节点分配给openstack的cpu核心数目，本次部署使用的物理服务器核心为2（CPU）*4（core），共有8个核心，openvswitch和计算节点自身也需要占用cpu资源因此不能完全分配给openstack，同时考虑到dpdk策略下vswitch的核心独占，该项数值与compute_ovs_pmd_cpu_mask和compute_ovs_dpdk_lcore_mask是互斥的；
- compute_hugepages_size：计算节点的大页内存配置，默认一个页面是2M；
- compute_hugepages_count：大页内存页面个数，本次部署中计算节点的物理内存是32G，本次分配给计算资源的大页内存总数为8192*2M=16G内存；
- compute_kernel_isolcpu：计算节点cpu核心隔离，设置计算节点的内核*不要使用*这些核心；
- compute_dpdk_driver：dpdk使用的内核模块；
- compute_ovs_pmd_cpu_mask：为了保证转发性能需要给ovs的PMD分配核心独占，CPU核心的分配采用掩码的方式，如本文中将cpu7分配给OVS，则掩码为0x80，cpu数从0开始计算，同时尽量**将DPDK和OVS分配的核心在同一个NUMA节点上**，关于OVS下的DPDK配置可以查阅官方手册[2];
- compute_ovs_dpdk_socket_mem：分配给dpdk的大页内存数，每个NUMA节点各4个G；
- compute_ovs_dpdk_lcore_mask：ovs中dpdk占用核心，同样采用掩码计算方式，最好与dpdk网卡所在NUMA节点一致，可以通过查看`/sys/bus/pci/devices/`目录下对应网卡总线的numa_node值查看，如dpdk网卡总线值为0000:42:00.0则使用命令`cat /sys/bus/pci/devices/0000\:42\:00.0/numa_node`查看所在NUMA节点；
- compute_ovs_memory_channels：内存通道，对应的物理服务器内存所使用的通道数；


然后修改dpdk网卡所使用的驱动，配置文件`mcp/reclass/classes/cluster/baremetal-mcp-ocata-ovs-dpdk-ha/openstack/compute.yml`默认使用的是`igb_uio`[3]由于部署的系统中没有`igb_uio`驱动，因此改为使用`uio_pci_generic`驱动。

配置完上述文件后使用`sudo ci/deploy.sh -b file:///home/opnfv/fuel/mcp/config/ -l bii -p pod1 -s os-nosdn-ovs-ha -B br-pxe,br-ctl -D `命令部署即可，推荐使用下述命令部署：

```shell
sudo nohup ci/deploy.sh -b file:///home/opnfv/fuel/mcp/config/ -l bii -p pod1 -s os-nosdn-ovs-ha -B br-pxe,br-ctl -D > opnfv_install_date +%Y-%m-%d`.log 2>&1 &
```

部署过程不占用终端，还可以通过`tail -f opnfv_install_date[date].log `查看部署过程。

【NOTE】当前的部署完成后存在一个BUG，网络服务的外网不正常，通过`service networking status`参看是否有错误。创建的虚拟机要想访问外网`float-to-ex`网桥需要存在，或者使用`route -n`查看是否存在能够访问外网的网关。

```shell
root@cmp001:~# brctl show
bridge name	bridge id		STP enabled	interfaces
br-ctl		8000.44a84214ee66	no		eno3.101
br-ex		8000.1efd4c8920ac	no		eno2
							           float-to-ex
virbr0		8000.525400b2f405	yes		virbr0-nic

root@cmp001:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.20.1    0.0.0.0         UG    0      0        0 br-ex
10.20.0.0       0.0.0.0         255.255.255.0   U     0      0        0 eno1
192.168.20.0    0.0.0.0         255.255.255.0   U     0      0        0 br-ex
192.168.101.0   0.0.0.0         255.255.255.0   U     0      0        0 br-ctl
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```

如不存在则需要重启网络服务`service networking restart`

错误1：

重启网络失败，通过`journalctl -xe`查看错误原因

![重启网络失败](https://raw.githubusercontent.com/louielong/blogPic/master/imgQfUS2Rr.jpg)

先将`/etc/network/if-up.d/route-br-ex`中的路由配置注释，然后使用`ip addr flush dev br-ctl`和`ip addr flush dev br-ex`（注意若使用的是`mgmt`网络访问会导致终端连接断开，在清除`br-ctl`的网址时请使用`pxe`网络的地址连接计算节点），随后使用`service networking restart`重启网络。

错误2:

创建虚拟机错误，查看`vim /var/log/nova/nova-compute.log`显示ovs权限不足，需要修改计算节点neutron插件中ovs配置`/etc/neutron/plugins/ml2/openvswitch_agent.ini`将`vhostuser_socket_dir = /var/run/openvswitch `改为`vhostuser_socket_dir = /var/run/openvswitch-vhost`

随后重启ovs服务`service neutron-openvswitch-agent restart`。



## 3 使能第二个DPDK网口

当前Fuel部署DPDK仅支持一块网卡配置，一般来说DPDK网卡中网口个数都是成对的，因此需要手动配置第二个DPDK网口。

### 3.1 控制节点neutron配置修改

#### 3.1.1 新增网络MTU修改

修改controller节点的`/etc/neutron/plugins/ml2/ml2_conf.ini`中

```ini
[ml2]
physical_network_mtus = physnet1:1500,physnet2:1500 
```

改为`physical_network_mtus = physnet1:1500,physnet2:1500,physnet3:1500`

#### 3.1.2 vlan配置

修改controller节点的`/etc/neutron/plugins/ml2/ml2_conf.ini`中

```ini
[ml2_type_vlan]
network_vlan_ranges = physnet2:1000:1030
```

改为`network_vlan_ranges = physnet2:1000:1030,physnet3:1031:1060`

**同时需要在物理交换机上配置1031~1060段的vlan支持。**

修改完成后使用`service neutron-server restart`重启neutron服务。

【Tips】因为需要修改三个控制节点，可以使用如下命令修改

```shell
sed -i 's/^physical_network_mtus =.*$/physical_network_mtus = physnet1:1500,physnet2:1500,physnet3:1500/g' /etc/neutron/plugins/ml2/ml2_conf.ini
sed -i 's/^network_vlan_ranges =.*$/network_vlan_ranges = physnet2:1000:1030,physnet3:1031:1060/g' /etc/neutron/plugins/ml2/ml2_conf.ini
```

配合saltstack命令可以更便捷的修改三个控制节点，登录到cfg01节点使用salt命令修改控制节点

```shell
root@cfg01:~# salt -C "ctl*" cmd.run "sed -i 's/^physical_network_mtus =.*$/physical_network_mtus = physnet1:1500,physnet2:1500,physnet3:1500/g' /etc/neutron/plugins/ml2/ml2_conf.ini"
root@cfg01:~# salt -C "ctl*" cmd.run "sed -i 's/^network_vlan_ranges =.*$/network_vlan_ranges = physnet2:1000:1030,physnet3:1031:1060/g' /etc/neutron/plugins/ml2/ml2_conf.ini"
root@cfg01:~# salt -C "ctl*" cmd.run "service neutron-server restart"
```

### 3.2 计算节点neutron配置修改

修改节点节点的`/etc/neutron/plugins/ml2/openvswitch_agent.ini`中ovs段

```ini
[ovs]
bridge_mappings = physnet1:br-floating,physnet2:br-prv 
```

改为`bridge_mappings = physnet1:br-floating,physnet2:br-prv,physnet3:br-prv1`，随后重启neutron服务`service neutron-openvswitch-agent restart`

[Tips]使用如下命令修改，同样的配合saltstack的命令可以更便捷的修改两个控制节点

```shell
sed -i 's/^bridge_mappings = .*$/bridge_mappings = physnet1:br-floating,physnet2:br-prv,physnet3:br-prv1/g' /etc/neutron/plugins/ml2/openvswitch_agent.ini
```

重启完neutron服务后通过`ovs-vsctl show`可以看到已经生成了一个br-prv1的网桥，如果没有生成，在控制节点上检查网络代理服务是否正常

```shell
root@ctl01:~# openstack network agent list
+--------------------------------------+--------------------+--------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host   | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+--------+-------------------+-------+-------+---------------------------+
| 03ee7d4d-f17a-4ca2-b864-71decdd58bac | Metadata agent     | cmp002 | None              | True  | UP    | neutron-metadata-agent    |
| 63b23081-2844-4712-a9d9-b5da80596558 | Open vSwitch agent | cmp001 | None              | True  | UP    | neutron-openvswitch-agent |
| 69acde22-0890-4e90-9f26-c20e69dace43 | DHCP agent         | cmp001 | nova              | True  | UP    | neutron-dhcp-agent        |
| 73ed4645-c22d-4150-a7aa-bd59474b5f59 | L3 agent           | cmp002 | nova              | True  | UP    | neutron-l3-agent          |
| b8e1e749-2254-48d7-ae46-54a6223971d1 | Open vSwitch agent | cmp002 | None              | True  | UP    | neutron-openvswitch-agent |
| bb0329cf-021b-42b4-a0b0-749889f5c619 | Metadata agent     | cmp001 | None              | True  | UP    | neutron-metadata-agent    |
| e0c05cb5-16c8-4da4-8eb2-d22087b69402 | DHCP agent         | cmp002 | nova              | True  | UP    | neutron-dhcp-agent        |
| ed1ed0f4-d3da-4486-828a-40f4a1759a7a | L3 agent           | cmp001 | nova              | True  | UP    | neutron-l3-agent          |
+--------------------------------------+--------------------+--------+-------------------+-------+-------+---------------------------+
```

br-prv1成功创建后需要手动添加dpdk网卡，首先为第二个网卡加载驱动

```shell
root@cmp001:~# ifconfig enp66s0f1 down
root@cmp001:~# dpdk-devbind -b uio_pci_generic 42:00.1
root@cmp001:~# dpdk-devbind -s

Network devices using DPDK-compatible driver
============================================
0000:42:00.0 '82599ES 10-Gigabit SFI/SFP+ Network Connection' drv=uio_pci_generic unused=ixgbe
0000:42:00.1 '82599ES 10-Gigabit SFI/SFP+ Network Connection' drv=uio_pci_generic unused=ixgbe
```

随后在ovs中配置dpdk网卡

```shell
root@cmp001:~# ovs-vsctl add-port br-prv1 dpdk1 -- set interface dpdk0 type=dpdk options:dpdk-devargs=0000:42:00.1 options:n_rxq=2
root@cmp001:~# service openvswitch-switch restart
```



【NOTE】在创建虚拟机时需要在虚拟机类型中额外添加大页内存特性设置，如

```shell
root@ctl01:~# source keystonercv3
root@ctl01:~# openstack flavor create m1.tiny --ram 64 --disk 0 --vcpus 1 --public
root@ctl01:~# nova flavor-key m1.tiny set hw:mem_page_size=large
```




[返回文首](#jump0)

【参考链接】

1)[dpdk配置](https://github.com/opnfv/Fuel/blob/stable/euphrates/mcp/reclass/classes/cluster/virtual-mcp-ocata-ovs-dpdk-noha/openstack/init.yml)

2)[OVS中DPDK配置](http://docs.openvswitch.org/en/latest/intro/install/dpdk/?highlight=pmd)

3)[DPDK网卡驱动](https://github.com/opnfv/Fuel/blob/fe9be64738ff1a1091e7df5b04b391fb15d6abc0/mcp/reclass/classes/cluster/baremetal-mcp-ocata-ovs-dpdk-ha/openstack/compute.yml#L37)

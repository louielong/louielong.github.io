---
title: Docker开启IPv6
date: 2020-02-26 13:55:29
tags:
 - Docker
keywords:
 - Docker
 - IPv6
categories:
 - Docker
description:
 - Docker 开启IPv6配置
summary_img:
 - https://raw.githubusercontent.com/louielong/blogPic/master/imgdocker_logo_ipv6.png
---

## 一、序言

因需要在创建的容器内使用IPv6网络进行测试，详细记录一下Docker如何开始IPv6，以及一些调试的**奇淫技巧**。

## 二、 docker配置IPv6

方案一：直接使用宿主机的网络在容器启动时加入`--host`参数；

```shell
docker run -d --name=busybox --net=host busybox top
```

显然方案一，太low，这里就不在介绍了。



方案二：将宿主机IPv6网络下划分一个子段，通过nd代理容器流量；

1）查看宿主机IPv6地址

```shell
xdnsadmin@ubuntu:~$ ifconfig ens3
ens3      Link encap:Ethernet  HWaddr 52:54:00:44:85:34
          inet addr:10.253.1.125  Bcast:10.253.1.255  Mask:255.255.255.0
          inet6 addr: fe80::5054:ff:fe44:8534/64 Scope:Link
          inet6 addr: 2001:eb:8001:e01::125/64 Scope:Global
  ...
```

2）划分子段并配置docker

配置`/etc/docker/daemon.json`，重启容器进程

```shell
xdnsadmin@ubuntu:~$ sudo cat /etc/docker/daemon.json
{
   "registry-mirrors": ["https://registry.docker-cn.com"],
   "ipv6": true,
   "fixed-cidr-v6": "2001:eb:8001:e01:2::/120"
}
xdnsadmin@ubuntu:~$ sudo service docker restart
xdnsadmin@ubuntu:~$ ifconfig docker0
docker0   Link encap:Ethernet  HWaddr 02:42:97:70:d9:1e
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: 2001:eb:8001:e01:2::1/120 Scope:Global
          inet6 addr: fe80::42:97ff:fe70:d91e/64 Scope:Link
          inet6 addr: fe80::1/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:265 errors:0 dropped:0 overruns:0 frame:0
          TX packets:101 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:36482 (36.4 KB)  TX bytes:12592 (12.5 KB)
```

3）配置转发

配置NDP（邻居发现协议）代理。关于这部分内容，请参见 [Using NDP proxying](https://docs.docker.com/v17.09/engine/userguide/networking/default_network/ipv6/#using-ndp-proxying)。

```shell
xdnsadmin@ubuntu:~$ sudo ip -6 neigh add proxy 2001:eb:8001:e01:2::2 dev ens3
xdnsadmin@ubuntu:~$ sudo sysctl net.ipv6.conf.default.forwarding=1
xdnsadmin@ubuntu:~$ sudo sysctl net.ipv6.conf.all.forwarding=1
xdnsadmin@ubuntu:~$ sudo sysctl net.ipv6.conf.ens3.accept_ra=0
net.ipv6.conf.ens3.accept_ra = 0
xdnsadmin@ubuntu:~$ sudo sysctl net.ipv6.conf.ens3.proxy_ndp=1
net.ipv6.conf.ens3.proxy_ndp = 1
```

4）测试IPv6

启动ubuntu 18.04镜像测试是否有IPv6网络，由于18.04镜像内无`ifconfig`和`ip`命令，可以安装相应工具或在宿主机上ping测试。

```shell
xdnsadmin@ubuntu:~$ docker run -itd ubuntu:18.04
78604aa6c229
xdnsadmin@ubuntu:~$ docker inspect 78604aa6c229
....
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "91f147cc84bbe58b8d817e98866573768b70c090463066fc086735860e3e586e",
                    "EndpointID": "adb855e72644da1e5959e2718f06149308c83fce3c9471d251eaedf31c6a33dd",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "2001:eb:8001:e01:2::1",
                    "GlobalIPv6Address": "2001:eb:8001:e01:2::2",
                    "GlobalIPv6PrefixLen": 120,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
....
xdnsadmin@ubuntu:~$ ping6 2001:eb:8001:e01:2::2 -c 2
PING 2001:eb:8001:e01:2::2(2001:eb:8001:e01:2::2) 56 data bytes
64 bytes from 2001:eb:8001:e01:2::2: icmp_seq=1 ttl=64 time=0.263 ms
64 bytes from 2001:eb:8001:e01:2::2: icmp_seq=2 ttl=64 time=0.058 ms

--- 2001:eb:8001:e01:2::2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.058/0.160/0.263/0.103 ms

```

如需在镜像内测试，需要安装相应工具

```shell
xdnsadmin@ubuntu:~$ docker exec -it 78604aa6c229 bash
root@78604aa6c229:/# sed -i "s/archive.ubuntu.com/mirrors.aliyun.com/g" /etc/apt/sources.list && apt update && apt install -y iproute2 iputils-ping
.....
root@78604aa6c229:/# ip a
....
34: eth0@if35: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:eb:8001:e01:2::2/120 scope global nodad
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link
       valid_lft forever preferred_lft forever
root@78604aa6c229:/# ping6 240c::6666 -c 2
PING 240c::6666(240c::6666) 56 data bytes
64 bytes from 240c::6666: icmp_seq=1 ttl=62 time=0.644 ms
64 bytes from 240c::6666: icmp_seq=2 ttl=62 time=0.413 ms

--- 240c::6666 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.413/0.528/0.644/0.117 ms
```

**【奇淫技巧】**也可以在宿主机上测试通过namespace的方式测试

```shell
xdnsadmin@ubuntu:~$ docker ps -a
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                                              NAMES
78604aa6c229        ubuntu:18.04            "/bin/bash"              3 hours ago         Up 16 minutes                                                          epic_blackwell
xdnsadmin@ubuntu:~$ docker inspect -f '{{.State.Pid}}' epic_blackwell
10740
xdnsadmin@ubuntu:~$ sudo mkdir /var/run/netns/
xdnsadmin@ubuntu:~$ sudo ln -fs /proc/10740/ns/net /var/run/netns/epic_blackwell
xdnsadmin@ubuntu:~$ sudo ip netns exec epic_blackwell ip -c a
....
34: eth0@if35: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:eb:8001:e01:2::2/120 scope global nodad
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link
       valid_lft forever preferred_lft forever
xdnsadmin@ubuntu:~$ sudo ip netns exec epic_blackwell ping6 240c::6666 -c 2
PING 240c::6666(240c::6666) 56 data bytes
64 bytes from 240c::6666: icmp_seq=1 ttl=62 time=0.548 ms
64 bytes from 240c::6666: icmp_seq=2 ttl=62 time=0.414 ms

--- 240c::6666 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.414/0.481/0.548/0.067 ms
```

上述操作可以保存为一个脚本

```shell
id=docker inspect -f '{{.State.Pid}}' $1
sudo mkdir /var/run/netns/
sudo ln -fs /proc/$id/ns/net /var/run/netns/$1
```



方案二的一个缺点是每创建一个容器或重启机器都需要添加`ip -6 neigh add proxy <ipv6_addr> dev ens3`，对于自动化构建较为麻烦。

【Tips】:检查宿主机DNS配置，在配置过程中出现宿主机仅配置了IPv6的DNS导致容器在开启IPv6服务后容器无法解析域名的情况。

方案三：创建支持IPv6的网桥

IPv6地址需要依据宿主机地址段来修改。

```shell
xdnsadmin@ubuntu:~$ docker network create -d bridge --ipv6 --subnet "2001:eb:8001:e01:3::/120" --gateway="2001:eb:8001:e01:3::1" --subnet=172.30.0.0/16 --gateway=172.30.0.1 IPv6Net
xdnsadmin@ubuntu:~$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
018c6246979b        IPv6Net             bridge              local
91f147cc84bb        bridge              bridge              local
e3db5466ef0a        host                host                local
f3c3efd928a6        none                null                local
xdnsadmin@ubuntu:~$ docker run -itd --ip=172.30.0.2 --ip6="2001:eb:8001:e01:3::2" --network=IPv6Net ubuntu:18.04
6d9933a583a6c5890f951ff8ab81382fc315c4ac862833397dc047562e667a51
xdnsadmin@ubuntu:~$ docker exec -it 6d9933a583a bash
root@6d9933a583a6:/# ed -i "s/archive.ubuntu.com/mirrors.aliyun.com/g" /etc/apt/sources.list && apt update && apt install -y iproute2 iputils-ping
....
root@6d9933a583a6:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
41: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:1e:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.30.0.2/16 brd 172.30.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:eb:8001:e01:3::2/120 scope global nodad
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe1e:2/64 scope link
       valid_lft forever preferred_lft forever
root@6d9933a583a6:/# ping6 240c::6666
PING 240c::6666(240c::6666) 56 data bytes
^C
--- 240c::6666 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3023ms

root@6d9933a583a6:/# ping baidu.com
PING baidu.com (39.156.69.79) 56(84) bytes of data.
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=1 ttl=45 time=25.6 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=2 ttl=45 time=28.5 ms
64 bytes from 39.156.69.79 (39.156.69.79): icmp_seq=3 ttl=45 time=23.6 ms
^C
--- baidu.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2446ms
rtt min/avg/max/mdev = 23.696/25.947/28.536/1.998 ms
```

 容器仍然无法访问v6网络，需要向方案二一样添加配置NDP（邻居发现协议）代理

```shell
xdnsadmin@ubuntu:~$ sudo ip -6 neigh add proxy 2001:eb:8001:e01:3::2 dev ens3
[sudo] password for xdnsadmin:
xdnsadmin@ubuntu:~$ docker exec -it 6d9933a583a bash
root@6d9933a583a6:/# ping6 240c::6666
PING 240c::6666(240c::6666) 56 data bytes
64 bytes from 2001::6666: icmp_seq=1 ttl=62 time=320 ms
64 bytes from 2001::6666: icmp_seq=2 ttl=62 time=0.370 ms
64 bytes from 2001::6666: icmp_seq=3 ttl=62 time=0.447 ms

--- 240c::6666 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.370/107.216/320.833/151.050 ms
```

相比方案二方案三的好处是能够指定容器的IP，可以预先配置好NPD代理。同样都有的缺点是IPv6无法像IPv4一样仅暴漏端口，IPv6下地址全端口都开放。



【参考链接】

1）[docker + ipv6](https://www.v2ex.com/t/553822#reply0)

2）[Docker容器支持IPv6的方法](https://blog.csdn.net/taiyangdao/article/details/83066009)
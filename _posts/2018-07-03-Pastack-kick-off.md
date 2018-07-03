---
title: Packstack kick off
categories:
- 部署
tags:
- Packstack
---

![](https://ws3.sinaimg.cn/large/006tNc79gy1fswrvhfaw3j30sg0co74w.jpg)

## Preparation

镜像：[CentOS7 minimal](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1804.iso)

环境：

​	Openstack Queue 版

​	controller node: 4 cpus, 8 memory, 2 bridge NICs

​	compute node: 2 cpus, 4 memory, 2 bridge NICs

配置：

* 网卡配置：

  ```shell
  # Controller node
  # ifcfg-enp0s3 NIC 外网网卡
  TYPE=Ethernet
  BOOTPROTO=static
  IPADDR=172.16.208.39
  NETMASK=255.255.255.0
  GATEWAY=172.16.208.254
  NM_CONTROLLED=no
  
  DEFROUTE=yes
  PEERDNS=yes
  PEERROUTES=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=yes
  IPV6_AUTOCONF=yes
  IPV6_DEFROUTE=yes
  IPV6_PEERDNS=yes
  IPV6_PEERROUTES=yes
  IPV6_FAILURE_FATAL=no
  NAME=enp0s3
  DEVICE=enp0s3
  ONBOOT=yes
          
  # ifcfg-enp0s8 NIC 数据网网卡
  TYPE=Ethernet
  BOOTPROTO=static
  IPADDR=172.16.208.138
  NETMASK=255.255.255.0
  GATEWAY=172.16.208.254
  NM_CONTROLLED=no
  
  DEFROUTE=yes
  PEERDNS=yes
  PEERROUTES=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=yes
  IPV6_AUTOCONF=yes
  IPV6_DEFROUTE=yes
  IPV6_PEERDNS=yes
  IPV6_PEERROUTES=yes
  IPV6_FAILURE_FATAL=no
  NAME=enp0s8
  DEVICE=enp0s8
  ONBOOT=yes
  
  # Compute node
  # ifcfg-enp0s3 NIC node 管理网网卡
  TYPE=Ethernet
  BOOTPROTO=static
  IPADDR=172.16.208.141
  NETMASK=255.255.255.0
  GATEWAY=172.16.208.254
  NM_CONTROLLED=no
  
  DEFROUTE=yes
  PEERDNS=yes
  PEERROUTES=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=yes
  IPV6_AUTOCONF=yes
  IPV6_DEFROUTE=yes
  IPV6_PEERDNS=yes
  IPV6_PEERROUTES=yes
  IPV6_FAILURE_FATAL=no
  NAME=enp0s3
  UUID=3052d15f-9489-4fd8-8820-bce765f30252
  DEVICE=enp0s3
  ONBOOT=yes
  
  # ifcfg-enp0s8 NIC 数据网网卡
  TYPE=Ethernet
  BOOTPROTO=static
  IPADDR=172.16.208.150
  NETMASK=255.255.255.0
  GATEWAY=172.16.208.254
  NM_CONTROLLED=no
  
  DEFROUTE=yes
  PEERDNS=yes
  PEERROUTES=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=yes
  IPV6_AUTOCONF=yes
  IPV6_DEFROUTE=yes
  IPV6_PEERDNS=yes
  IPV6_PEERROUTES=yes
  IPV6_FAILURE_FATAL=no
  NAME=enp0s8
  DEVICE=enp0s8
  ONBOOT=yes
  ```

* Linux 环境配置

  **在 controller 和 compute 节点上做一下配置**

  ```shell
  # 配置环境变量 /etc/environment
  LANG=en_US.utf-8
  LC_ALL=en_US.utf-8
  
  # controller 和 compute 节点配置 yum 源， Queen 版
  sudo yum update -y
  sudo yum install -y centos-release-openstack-queens
  sudo yum update -y
  
  # 网络配置
  sudo systemctl disable firewalld
  sudo systemctl stop firewalld
  sudo systemctl disable NetworkManager
  sudo systemctl stop NetworkManager
  sudo systemctl enable network
  sudo systemctl start network
  ```

  

## Allinone

Packstack 的 allinone 安装非常简单只需要执行 `sudo packstack --allinone`。

Allinone 安装时，Neutron 将回环网卡作为外网网卡。

## With external network

#### 使用命令

Packstack 安装时，Neutron 使用外网网卡的命令也很简单，**注意这里使用网卡`enp0s3`作为外网网卡**：

`packstack --allinone --provision-demo=n --os-neutron-ovs-bridge-mappings=extnet:br-ex --os-neutron-ovs-bridge-interfaces=br-ex:enp0s3 --os-neutron-ml2-type-drivers=vxlan,flat`

* `--provision-demo`: 不用 demo 创建任何资源
* `--os-neutron-ovs-bridge-mappings` : 创建虚拟网络时，虚拟网络和 ovs 网桥的 mapping 关系。
* `--os-neutron-ovs-bridge-interfaces`：创建虚拟网络时，ovs 网桥和物理网卡的 mapping 关系。
* `--os-neutron-ml2-type-drivers` : 配置可以使用的二层网络类型。

#### 使用配置文件

除了使用命令行，还可以使用配置文件部署 Packstack：

生成 Packstack 配置文件: `sudo packstack --gen-answer-file=~/answers.cfg`

对应上面命令行中的参数，需要在 `~/answers.cfg` 中配置以下配置项：

* `CONFIG_PROVISION_DEMO=n`
* `CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=extnet:br-ex`
* `CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-ex:enp0s3`
* `CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vxlan,flat`

配置完以上配置后执行：

`packstack --answer-file=~/answers.cfg`

#### 安装完成后配置

在 Packstack 安装完成后，使用 `ip addr` 命令可以看到如下：

```shell
[root@localhost ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP group default qlen 1000
    link/ether 08:00:27:c5:30:43 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fec5:3043/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8b:da:87 brd ff:ff:ff:ff:ff:ff
    inet 172.16.208.138/24 brd 172.16.208.255 scope global dynamic enp0s8
       valid_lft 13349sec preferred_lft 13349sec
    inet6 fe80::a00:27ff:fe8b:da87/64 scope link
       valid_lft forever preferred_lft forever
4: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether aa:9a:1f:ae:6d:16 brd ff:ff:ff:ff:ff:ff
5: br-tun: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether da:41:c5:36:d5:4d brd ff:ff:ff:ff:ff:ff
8: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether a6:75:4c:23:26:49 brd ff:ff:ff:ff:ff:ff
16: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 08:00:27:c5:30:43 brd ff:ff:ff:ff:ff:ff
    inet 172.16.208.39/24 brd 172.16.208.255 scope global br-ex
       valid_lft forever preferred_lft forever
    inet6 fe80::acd5:cdff:fe4a:f44b/64 scope link
       valid_lft forever preferred_lft forever
```

可以看到 packstack 成功将生成 `br-ex`，`br-int` 和 `br-tun`，packstack 将 enp0s3 网卡的 IP 地址挂在了 `br-ex` 上。

再查看 `/etc/sysconfig/network-script` 文件夹下 `ifcfg-br-ex` 和 外网网卡配置文件 `ifcfg-enp0s3`

```shell
[root@localhost network-scripts]# cat ifcfg-br-ex
IPADDR=172.16.208.39
NETMASK=255.255.255.0
GATEWAY=172.16.208.254
NM_CONTROLLED=no
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
UUID=9d923113-43df-4c9b-b3d5-8b248e95f263
ONBOOT=yes
DEVICE=br-ex
NAME=br-ex
DEVICETYPE=ovs
OVSBOOTPROTO=static
TYPE=OVSBridge
OVS_EXTRA="set bridge br-ex fail_mode=standalone"

# 注意如果 ifcfg-enp0s3 文件与下面配置不符合，将 ifcfg-enp0s3 配置成如下配置
[root@localhost network-scripts]# cat ifcfg-enp0s3
DEVICE=enp0s3
NAME=enp0s3
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-ex
ONBOOT=yes
BOOTPROTO=none
```

Packstack 在 Linux 上添加 `ifcfg-br-ex` 网卡，同时将外网网卡 enp0s3 挂在了 `ifcfg-br-ex` 网桥下。

## Adding compute nodes and lbass

#### 使用命令行

这里记录下使用配置文件的方式添加 compute node，使用命令行方式添加可以参考[Creating an OpenStack development environment with an existing External Network via Packstack](https://medium.com/@jmarhee/creating-an-openstack-development-environment-with-an-existing-external-network-via-packstack-651df38f4b57) 这篇文。

使用命令行安装 lbass 服务，在命令行中添加：`--os-neutron-lbaas-install=y`

#### 使用配置文件

在基于以上 **with external network** 的使用配置文件的配置基础上，还需要在配置文件中添加以下配置：

* `CONFIG_CONTROLLER_HOST=172.16.208.39` : 指定 controller node。
* `CONFIG_COMPUTE_HOSTS=172.16.208.141` ：指定 compute node，可以是多个 nodes，用逗号隔开。
* `CONFIG_NETWORK_HOSTS=172.16.208.39` ：指定 network node。
* `CONFIG_NEUTRON_OVS_TUNNEL_IF=enp0s8` : 注意这里的 `enp0s8` 网卡名称是 compute 节点上的网卡名称(compute 节点上必须有同名网卡，但是 controller 节点上貌似也需要有同名网卡)，指定 compute（controller？） 上用于建立 vxlan tunnel 的物理网卡。
* `CONFIG_LBAAS_INSTALL=y` ：配置安装 lbass。

配置完成后执行命令 `packstack --answer-file=~/answers.cfg`。

#### 安装完成后的配置

使用 `ip addr` 分别在 costroller 和 compute 可以查看到如下：

```shell
# controller 节点上多了一个 vxlan_sys_4789 网卡
[root@localhost network-scripts]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP group default qlen 1000
    link/ether 08:00:27:c5:30:43 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::a00:27ff:fec5:3043/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8b:da:87 brd ff:ff:ff:ff:ff:ff
    inet 172.16.208.138/24 brd 172.16.208.255 scope global dynamic enp0s8
       valid_lft 11077sec preferred_lft 11077sec
    inet6 fe80::a00:27ff:fe8b:da87/64 scope link
       valid_lft forever preferred_lft forever
4: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether aa:9a:1f:ae:6d:16 brd ff:ff:ff:ff:ff:ff
5: br-tun: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether da:41:c5:36:d5:4d brd ff:ff:ff:ff:ff:ff
8: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether a6:75:4c:23:26:49 brd ff:ff:ff:ff:ff:ff
15: vxlan_sys_4789: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65000 qdisc noqueue master ovs-system state UNKNOWN group default qlen 1000
    link/ether b6:26:b9:70:3b:8a brd ff:ff:ff:ff:ff:ff
    inet6 fe80::b426:b9ff:fe70:3b8a/64 scope link
       valid_lft forever preferred_lft forever
16: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 08:00:27:c5:30:43 brd ff:ff:ff:ff:ff:ff
    inet 172.16.208.39/24 brd 172.16.208.255 scope global br-ex
       valid_lft forever preferred_lft forever
    inet6 fe80::acd5:cdff:fe4a:f44b/64 scope link
       valid_lft forever preferred_lft forever

# compute 节点上也多了一个 vxlan_sys_4789 网卡
[root@localhost ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ec:03:b0 brd ff:ff:ff:ff:ff:ff
    inet 172.16.208.141/24 brd 172.16.208.255 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:feec:3b0/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:7c:88:51 brd ff:ff:ff:ff:ff:ff
    inet 172.16.208.150/24 brd 172.16.208.255 scope global dynamic enp0s8
       valid_lft 11149sec preferred_lft 11149sec
    inet6 fe80::a00:27ff:fe7c:8851/64 scope link
       valid_lft forever preferred_lft forever
4: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3e:8c:ea:fa:c1:db brd ff:ff:ff:ff:ff:ff
5: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 96:76:46:dd:ee:43 brd ff:ff:ff:ff:ff:ff
6: br-tun: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d2:3e:61:4d:0b:40 brd ff:ff:ff:ff:ff:ff
7: vxlan_sys_4789: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65000 qdisc noqueue master ovs-system state UNKNOWN group default qlen 1000
    link/ether 6e:b4:2c:66:61:38 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::6cb4:2cff:fe66:6138/64 scope link
       valid_lft forever preferred_lft forever
```

从以上打印可以看出 controller 和 compute 节点上都多处了 vxlan_sys_4789 网卡。同时查看 br-tun，可以看出在 compute 和 controller 节点上的 ovs br-tun 都被配置上了 vtep。（但是 vtep 上所绑定的 port `vxlan-ac10d096` 和新添加的网卡 `vxlan_sys_4789` 名称并不一致？）

```shell
# 注意这些配置中使用的 IP 地址都是 controller 和 compute 各节点上物理网卡 enp0s8 上的 IP 地址
# controller 节点上的 ovs
[root@localhost network-scripts]# ovs-vsctl show
8690bb97-ba2f-4b7b-8087-8d3484ba2d64
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port "tap20baa8bd-cb"
            Interface "tap20baa8bd-cb"
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "qg-f7550344-09"
            tag: 2
            Interface "qg-f7550344-09"
                type: internal
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
        Port "qr-4f35d33c-99"
            tag: 1
            Interface "qr-4f35d33c-99"
                type: internal
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-ac10d096"
            Interface "vxlan-ac10d096"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.16.208.138", out_key=flow, remote_ip="172.16.208.150"}
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-ex
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-ex
            Interface br-ex
                type: internal
        Port "enp0s3"
            Interface "enp0s3"
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
    ovs_version: "2.9.0"
    
# compute 节点上的 ovs
[root@localhost ~]# ovs-vsctl show
d5386df5-93a2-4ff3-96fd-82fe1de3954f
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-ac10d08a"
            Interface "vxlan-ac10d08a"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.16.208.150", out_key=flow, remote_ip="172.16.208.138"}
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
    ovs_version: "2.9.0"
```

注意这些配置中使用的 `IP` 地址都是 controller 和 compute 各节点上物理网卡 `enp0s8` 上的 IP 地址。

在 controller 节点（网络节点）上检查 LBasS 正常运行：

```shell
[root@localhost ~]# systemctl status neutron-lbaasv2-agent.service
● neutron-lbaasv2-agent.service - OpenStack Neutron Load Balancing as a Service (API v2.x) Agent
   Loaded: loaded (/usr/lib/systemd/system/neutron-lbaasv2-agent.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2018-07-03 02:33:00 EDT; 2h 22min ago
 Main PID: 354 (neutron-lbaasv2)
    Tasks: 1
   CGroup: /system.slice/neutron-lbaasv2-agent.service
           └─354 /usr/bin/python2 /usr/bin/neutron-lbaasv2-agent --config-file /usr/share/neutron/neutron-dist.conf --config-file /u...

Jul 03 02:33:00 localhost.localdomain systemd[1]: Started OpenStack Neutron Load Balancing as a Service (API v2.x) Agent.
Jul 03 02:33:00 localhost.localdomain systemd[1]: Starting OpenStack Neutron Load Balancing as a Service (API v2.x) Agent...
```

to be continue ...

## Reference

Packstack 初始化配置：https://www.rdoproject.org/install/packstack/

Packstack 添加一个 compute node：https://www.rdoproject.org/install/adding-a-compute-node/

Packstack LBasS 安装：https://www.rdoproject.org/networking/lbaas/

M 版 LBasS 一些功能：https://docs.openstack.org/mitaka/networking-guide/config-lbaas.html


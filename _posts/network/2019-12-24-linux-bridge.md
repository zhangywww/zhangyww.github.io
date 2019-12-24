---
layout: post
title: linux bridge
tag: network
---

# brctl

bridge是二层网络设备。bridge建立于linux内核之中。我们可以通过btrctl命令来创建和删除software bridge。通过**brctl**命令创建的bridge是**临时的**，在系统重启后会自动删除，如果需要创建永久的bridge，需要通过编辑配置文件**/etc/sysconfig/network-scripts/ifcfg-\***来创建(CentOS)。

## brctl的安装 

在CentOS下通过yum安装**bridge-utils**进行安装。

```shell
yum install bridge-utils
```

## brctl 创建bridge

下面命令创建一个名为**br0**的bridge。

```shell
brctl addbr br0
```

## brctl 删除bridge

下面命令删除一个名为**br0**的bridge。

```shell
brctl delbr br0
```

## bridge增加interface

下面命令将interface **eth0** 和 **eth1** 增加进bridge **br0**。

```shell
brctl addif br0 eth0
brctl addif br0 eth1
```

## bridge移除interface

下面命令将interface **eth0** 和 **eth1** 从bridge **br0** 移除。

```shell
brctl delif br0 eth0
brctl delif br0 eth1
```

# 创建永久bridge

通过配置文件**/etc/sysconfig/network-scripts/ifcfg-\***可以创建永久的bridge (CentOS)。

## interface接口文件

通过下面的配置文件可以配置**eth0**接口和**eth1**接口。

配置**eth0**。
```shell
# /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE=eth0
TYPE=Ethernet
BRIDGE=br0
```

配置**eth1**。
```shell
# /etc/sysconfig/network-scripts/ifcfg-eth1

DEVICE=eth1
TYPE=Ethernet
BRIDGE=br0
```

## 创建以DHCP分配IP的bridge

下面的配置文件创建通过DHCP分配IP的bridge。
```shell
# /etc/sysconfig/network-scripts/ifcfg-br0

DEVICE=br0
TYPE=Bridge
ONBOOT=yes
DELAY=0
BOOTPROTO=dhcp
ONBOOT=yes
```

## 创建静态配置IP的bridge

下面的配置文件创建静态分配IP的bridge。

```shell
# /etc/sysconfig/network-scripts/ifcfg-br0

DEVICE=br0
TYPE=Bridge
BOOTPROTO=static
IPADDR=<static_IP_address>
NETMASK=<netmask>
GETEWAY=<gateway>
ONBOOT=yes
```

## 创建transparent bridge

下面的配置文件在两个interface间创建透明的bridge(transparent bridge)。

```shell
# /etc/sysconfig/network-scripts/ifcfg-br0

DEVICE=br0
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
```
---
title: 静态ip地址
comments: false
toc: false
date: 2020-02-29 14:30:53
categories: linux
tags:
---

## ubuntu

该方法ubuntu 18.04失效

修改配置文件 */etc/network/interfaces*

``` txt
auto eth0                  #设置自动启动eth0接口
iface eth0 inet static     #配置静态IP
address 192.168.11.88      #IP地址
netmask 255.255.255.0      #子网掩码
gateway 192.168.11.1       #默认网关
```

### 可选配置

**动态IP无法设置虚拟网卡,如果为动态IP则不用配置**
编辑 */etc/resolv.conf*,往配置文件中添加上面配置的网段的网关，我们这里上面配置了三个网段，那么我们的配置文件中添加以下信息:

``` sh
nameserver 172.16.254.254
nameserver 192.168.8.1
nameserver 192.168.88.1
```

重启网络，使配置生效`$ sudo /etc/init.d/networking restart`
查看ip是否配置成功 ` $ ifconfig`

## ubuntu 18.04

Netplan是Ubuntu 17.10中引入的一种新的命令行网络配置实用程序，用于在 Ubuntu 系统中轻松管理和配置网络设置。 它允许您使用 YAML 格式的描述文件来抽像化定义网络接口的相关信息。

Netplan可以使用 NetworkManager 或 Systemd-networkd 的网络守护程序来做为内核的接口。Netplan 的默认描述文件在 /etc/netplan/*.yaml 里，Netplan 描述文件采用了 YAML 语法。

在 Ubuntu 18.04 中如果再通过原来的 ifupdown 工具包继续在 /etc/network/interfaces 文件里配置管理网络接口是无效的。

1. 查看网关地址`route -n`

``` sh
user@ubuntu:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 enp0s3
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 enp0s3
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 enp0s3
```

2. 使用Netplan配置静态ip

修改 */etc/netplan/50-cloud-init.yaml* 文件 ，桌面版的是 */etc/netplan/01-network-manager-all.yaml* 将

``` yaml
network:
    ethernets:
        ens33:
            dhcp4: true
    version: 2
```

修改为

``` yaml
network:
    ethernets:
        ens33:
            addresses: [192.168.36.202/24]
            gateway4: 192.168.36.1
            dhcp4: no
            nameservers:
                    addresses: [114.114.114.114,180.76.76.76]
    version: 2
```

> ens33 是网卡名称
>
> gateway4 配置为`route -n`查看到的网关
>
> dhcp4 no代表不是用dhcp动态获取ip，yes代表使用dhcp动态获取ip
>
> nameservers dns地址

使得配置文件生效`$ netplan apply`

### 可选配置

解决resolv.conf配置文件被覆盖,首先安装resolvconf软件`sudo resolvconf -u`生成base head original tail 四个文件，head提示不可编辑，所以修改base和tail两个文件
*/etc/resolvconf/resolv.conf.d/base*(如果没有这个文件的手动创建),*/etc/resolvconf/resolv.conf.d/tail* 

reboot重启机器后查看/etc/resolv.conf已经有添加的dns配置了


## centos

修改 */etc/sysconfig/network-scripts/ifcfg-网卡* 文件

``` sh
BOOTPROTO=static          #静态ip
IPADDR=192.168.1.20       #IP地址
NETMASK=255.255.255.0     #子网掩码
NETWORK=192.168.1.1       #默认网关
DNS1=192.168.1.1          #DNS服务器
ONBOOT=yes                #是否激活网卡
```

### 可选配置

**动态IP无法设置虚拟网卡,如果为动态IP则不用配置**
编辑 */etc/resolv.conf*,往配置文件中添加上面配置的网段的网关，我们这里上面配置了三个网段，那么我们的配置文件中添加以下信息:

``` sh
nameserver 172.16.254.254
nameserver 192.168.8.1
nameserver 192.168.88.1
```

使得配置生效 `systemctl restart network`(centos7) `service network restart`(centos6)



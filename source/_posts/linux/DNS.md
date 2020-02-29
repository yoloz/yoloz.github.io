---
title: DNS
comments: false
toc: false
date: 2020-02-29 14:54:26
categories: linux
tags:
---

## unknown host error

1. 确定设置了域名服务器(dns) *cat /etc/resolv.conf* 

``` sh
nameserver 119.29.29.29  
nameserver 182.254.116.116  
```

> Public DNS+(首选：119.29.29.29备选：182.254.116.116)
>
> 114DNS(首选：114.114.114.114备选：114.114.114.115)
>
>ubuntu可能修改此文件无效,详情见下文；
>
>配置好保存即生效,无需重启网络；

2. 确保路由表正常

``` sh
[root@CentOS5 ~]# netstat -rn  
Kernel IP routing table  
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface  
192.168.128.0   0.0.0.0         255.255.255.0   U         0 0          0 eth0  
169.254.0.0     0.0.0.0         255.255.0.0     U         0 0          0 eth0  
0.0.0.0         192.168.128.2   0.0.0.0         UG        0 0          0 eth0 
```

如果未设置, 则增加网关`route add default gw 192.168.128.2`

**对于unknown host问题，除了上述方式，最简单的是查询出地址对应的ip，添加进hosts文件即可**

## 查看DNS

* 查看文件`cat /etc/resolv.conf`

* 使用nslookup

``` sh
#nslookup www.baidu.com

Server:		127.0.0.53
Address:	127.0.0.53#53
#上述地址即为DNS服务器IP

Non-authoritative answer:
www.baidu.com	canonical name = www.a.shifen.com.
Name:	www.a.shifen.com
Address: 180.101.49.12
Name:	www.a.shifen.com
Address: 180.101.49.11
```

* 使用dig

``` sh
#dig |grep SERVER
;; SERVER: 127.0.0.53#53(127.0.0.53)
```






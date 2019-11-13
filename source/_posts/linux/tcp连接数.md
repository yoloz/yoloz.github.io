---
title: tcp连接数
comments: false
toc: false
date: 2019-11-13 09:48:00
categories:
tags:
---

在Linux平台上，无论编写客户端程序还是服务端程序，在进行高并发TCP连接处理时，最高的并发数量都要受到系统对用户单一进程同时可打开文件数量的限制（系统为每个TCP连接都要创建一个socket句柄，每个socket句柄同时也是一个文件句柄）. 可使用 `ulimit -n` 查看系统允许当前用户单个进程打开的文件数限制，排除每个进程必然打开的标准输入，标准输出，标准错误，服务器监听socket, 进程间通讯的unix域socket等文件，实际能够并发的TCP连接数要比限制值少。

> [ulimit](/ulimit)

## 修改网络内核对TCP连接的有关限制

在Linux上编写支持高并发TCP连接的客户端通讯处理程序时，有时会发现尽管已经解除了系统对用户同时打开文件数的限制，但仍会出现并发TCP连接数增加到一定数量时，再也无法成功建立新的TCP连接的现象, 出现这种现在的原因有多种:

**可能是因为Linux网络内核对本地端口号范围有限制**, 进一步分析为什么无法建立TCP连接，会发现问题出在connect()调用返回失败，查看系统错误提示消息是"Can't assign requested address", 同时用tcpdump工具监视网络会发现根本没有TCP连接时客户端发SYN包的网络流量, 这些情况说明问题在于本地Linux系统内核中有限制。

> 根本原因在于Linux内核的TCP/IP协议实现模块对系统中所有的客户端TCP连接对应的本地端口号的范围进行了限制（如内核限制本地端口号的范围为1024~32768之间）。当系统中某一时刻同时存在太多的TCP客户端连接时，由于每个TCP客户端连接都要占用一个唯一的本地端口号（此端口号在系统的本地端口号范围限制中），如果现有的TCP客户端连接已将所有的本地端口号占满，则此时就无法为新的TCP客户端连接分配一个本地端口号了，因此系统会在这种情况下在connect()调用中返回失败，并将错误提示消息设为"Can't assign requested address".

增大本地端口范围限制:  
1, 修改/etc/sysctl.conf文件，在文件中添加如下行： `net.ipv4.ip_local_port_range = 1024 65000`

> 将系统对本地端口范围限制设置为1024~65000之间。注意: 本地端口范围的最小值必须大于或等于1024, 最大值则应小于或等于65535

2, 执行 `sysctl -p` ，如果系统没有错误提示，就表明新的本地端口范围设置成功。如果按上述端口范围进行设置，则理论上单独一个进程最多可以同时建立60000多个TCP客户端连接

**可能是因为Linux网络内核的防火墙对最大跟踪的TCP连接数有限制**，此时程序会表现为在connect()调用中阻塞，如同死机，如果用tcpdump工具监视网络，也会发现根本没有TCP连接时客户端发SYN包的网络流量。

> 防火墙在内核中会对每个TCP连接的状态进行跟踪，跟踪信息将会放在位于内核内存中的conntrackdatabase中，这个数据库的大小有限，当系统中存在过多的TCP连接时，数据库容量不足，IP_TABLE无法为新的TCP连接建立跟踪信息，于是表现为在connect()调用中阻塞.

增大内核对最大跟踪的TCP连接数的限制:
1, 修改/etc/sysctl.conf文件，在文件中添加如下行： `net.ipv4.ip_conntrack_max = 10240`

> 将系统对最大跟踪的TCP连接数限制设置为10240. 注意: 此限制值要尽量小，以节省对内核内存的占用

2, 执行 `sysctl -p` ，如果系统没有错误提示，就表明系统对新的最大跟踪的TCP连接数限制修改成功。如果按上述参数进行设置，则理论上单独一个进程最多可以同时建立10000多个TCP客户端连接

## 内核参数sysctl.conf的优化

/etc/sysctl.conf是用来控制linux网络的配置文件，对于依赖网络的程序（如web服务器和cache服务器）非常重要，推荐配置如下:

``` sh
#cp /etc/sysctl.conf /etc/sysctl.conf.bak
#echo ""> /etc/sysctl.conf
#vim /etc/sysctl.conf
net.ipv4.ip_local_port_range = 1024 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_window_scaling = 0
net.ipv4.tcp_sack = 0
net.core.netdev_max_backlog = 30000
net.ipv4.tcp_no_metrics_save = 1
net.core.somaxconn = 10240
net.ipv4.tcp_syncookies = 0
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
```

> 这个配置参考于cache服务器varnish的推荐配置和SunOne服务器系统优化的推荐配置
> 可能"net.ipv4.tcp_fin_timeout = 3"的配置会导致页面经常打不开，可以调整"net.ipv4.tcp_fin_timeout = 10", 在10s的情况下一般没问题了
> 修改后执行 `sysctl -p` 生效

linux系统优化完网络必须调高系统允许打开的文件数才能支持大的并发，如：

``` sh
echo "ulimit -HSn 65536" >> /etc/rc.local
echo "ulimit -HSn 65536" >> /root/.bash_profile
```

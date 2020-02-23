---
title: alternatives
comments: false
toc: false
date: 2019-07-24 16:10:31
categories: linux
tags:
---
alternatives用于存放系统的一些默认打开程序的信息和配置, 比如默认的编辑器、默认的网络浏览器、 默认的图形登陆器、默认的鼠标指针等  

> `alternatives` 和 `update-alternatives` 其实一个东东，都指向alternatives

就常用的java而言，/usr/bin/java连接到/etc/alternatives/java，而/etc/alternatives/java连接到具体的jdk,

``` bash
ll /usr/bin/java
lrwxrwxrwx 1 root root 22 6月  28 16:21 /usr/bin/java -> /etc/alternatives/java*

ll /etc/alternatives/java
lrwxrwxrwx 1 root root 20 7月  24 13:23 /etc/alternatives/java -> /opt/jdk-12/bin/java*
```

## install

将不同版本jdk注册到alternatives中:

``` bash
sudo update-alternatives --install /usr/bin/java  java /opt/jdk-12/bin/java 1

sudo update-alternatives --install /usr/bin/java  java /opt/jdk-11.0.2/bin/java 2

sudo update-alternatives --install /usr/bin/java  java /opt/jdk1.8.0_152/bin/java 3
```

## display

 `update-alternatives --display editor` 查看机器上的所有可以用来被 editor链接的命令  

``` bash
sudo update-alternatives --display java
java - manual mode
  link best version is /opt/jdk1.8.0_152/bin/java
  link currently points to /opt/jdk1.8.0_152/bin/java
  link java is /usr/bin/java
/opt/jdk-11.0.2/bin/java - priority 2
/opt/jdk-12/bin/java - priority 1
/opt/jdk1.8.0_152/bin/java - priority 3
```

## config

配置默认使用的jdk:

``` bash
sudo update-alternatives --config java
There are 3 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                        Priority   Status
------------------------------------------------------------
  0            /opt/jdk1.8.0_152/bin/java   3         auto mode
  1            /opt/jdk-11.0.2/bin/java     2         manual mode
* 2            /opt/jdk-12/bin/java         1         manual mode
  3            /opt/jdk1.8.0_152/bin/java   3         manual mode

Press <enter> to keep the current choice[*], or type selection number: 3
update-alternatives: using /opt/jdk1.8.0_152/bin/java to provide /usr/bin/java (java) in manual mode
```

## remove  

``` bash
update-alternatives --remove name path
#update-alternatives --remove java /opt/jdk-11.0.2/bin/java
```

> name 是一个在/etc/alternatives中的名字即link，而 path是希望删除的可选程序名的绝对路径名（只是从列表中删除了这个程序，并不会真的从硬盘上删除程序的可执行文件）。如果从一个alternative组中删除了一个正在被链接的程序并且这个组仍然没有变成空的，update-alternatives会自动用一个具有其他优先级的可选程序代替原来的程序。如果这个组变成空的了，那么连这个alternative 组都会被移除

---
title: ssh
comments: false
toc: false
date: 2020-02-26 18:38:30
categories: linux
tags:
---

## ssh免密登录

* 客户端生成密钥

`ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa`

>生成密钥过程中，建议采用默认值，只需要按三次回车之后，就会在～/.ssh目录下生成密钥文件，其中id_rsa为私钥，id_rsa.pub为公钥

* 服务器配置

服务器的~/.ssh/authorized_keys文件保存可快速连接的客户端的公钥。需要把客户端生成的id_rsa.pub文件的内容拷贝到authorized_keys文件的末尾。

``` sh
scp ~/.ssh/id_rsa.pub user@server:/home/user
ssh user@server -p port
cat ~/id_dsa.pub >> ~/.ssh/authorized_keys
```

>如果还是需要密码，则是由于目录及权限问题造成
chown username: /home/username/.ssh
chown username: /home/username/.ssh/*
chmod 700 /home/username/.ssh
chmod 600 /home/username/.ssh/*

* 在客户端配置服务器登录相关参数

除了密码之外，登录时，还需要配置ip地址、端口、用户等信息，也比较繁琐。可通过客户端的~/.ssh/config配置服务器的相关参数简化登录命令。

config文件的配置内容如下：

``` txt
Host server
HostName 192.168.1.1
Port 22
User ubuntu
```

>Host为服务器的名称，输入登录命令时使用
>
>HostName为服务器的ip地址
>
>Port为ssh的端口
>
>User为服务器的用户名

**从此以后，登录服务器如此`ssh server`**

## ssh拷贝文件

遇到ssh能连接但是scp被禁止的情形可以用如下方式拷贝文件到本地`ssh root@10.68.120.214 'cd /root && cat shandongReq2Join.tar.gz' > ~/shandongReq2Join.tar.gz`
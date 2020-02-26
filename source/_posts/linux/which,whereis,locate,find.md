---
title: which, whereis, locate, find
comments: false
toc: false
date: 2019-09-18 23:54:00
categories: linux
tags:
---

## which

只能寻找可执行文件, 并在PATH变量里面寻找  

## whereis

从linux文件数据库（/var/lib/slocate/slocate.db）寻找，有可能找到刚刚删除，或者没有发现新建的文件, 全匹配模式

## locate

同上, 不过文件名是部分匹配

## find

是直接在硬盘上搜寻，功能强大

``` sh
find .  #列出当前目录及子目录下所有文件和文件夹

find /home -name "*.txt" #在/home目录下查找以.txt结尾的文件名

find /home -iname "*.txt" #同上，但忽略大小写

#当前目录及子目录下查找所有以.txt和.pdf结尾的文件
find . \( -name "*.txt" -o -name "*.pdf" \)
#或
find . -name "*.txt" -o -name "*.pdf"  

find /home ! -name "*.txt" #找出/home下不是以.txt结尾的文件

find . -type f -name "*.txt" -delete #删除当前目录下所有.txt文件

find . -empty  #要列出所有长度为零的文件
```

* 根据文件类型进行搜索

`find . -type 类型参数`

类型参数列表：

* f 普通文件
* l 符号连接
* d 目录
* c 字符设备
* b 块设备
* s 套接字
* p Fifo

``` sh
find . -maxdepth 3 -type f  #向下最大深度限制为3
find . -mindepth 2 -type f  #搜索出深度距离当前目录至少2个子目录的所有文件
```

* 根据文件时间戳进行搜索

`find . -type f 时间戳`

UNIX/Linux文件系统每个文件都有三种时间戳：

* 访问时间（-atime/天，-amin/分钟）：用户最近一次访问时间
* 修改时间（-mtime/天，-mmin/分钟）：文件最后一次修改时间
* 变化时间（-ctime/天，-cmin/分钟）：文件数据元（例如权限等）最后一次修改时间

``` sh
find . -type f -atime -7  #搜索最近七天内被访问过的所有文件
find . -type f -atime 7   #搜索恰好在七天前被访问过的所有文件
find . -type f -atime +7  #搜索超过七天内被访问过的所有文件
find . -type f -amin +10  #搜索访问时间超过10分钟的所有文件
find . -type f -newer file.log  #找出比file.log修改时间更长的所有文件
```

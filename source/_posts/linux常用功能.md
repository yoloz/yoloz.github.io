---
title: linux常用功能
comments: false
toc: false
date: 2019-02-06 15:37:32
categories: linux
tags:
---

## 创建用户及组

创建用户组 `groupadd` ，创建用户 `useradd` , 设置密码 `passwd` ，配置sudo权限 `vim /etc/sudoers` 并添加 `userName ALL=(ALL) ALL` , 删除用户 `userdel [-r] userName` , 删除用户组 `groupdel`

> 删除用户时携带-r表示完全删除，不添加则home及相关数据保留仅删除用户

## 查看用户及组

用户列表文件 `/etc/passwd` , 用户组列表文件 `/etc/group`

## which、whereis、locate和find命令

 `which` 只能寻找可执行文件, 并在PATH变量里面寻找  
 `whereis` 从linux文件数据库（/var/lib/slocate/slocate.db）寻找，有可能找到刚刚删除，或者没有发现新建的文件, 全匹配模式
 `locate` 同上, 不过文件名是部分匹配
 `find` 是直接在硬盘上搜寻，功能强大

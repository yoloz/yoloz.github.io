---
title: user, group
comments: false
toc: false
date: 2019-09-18 23:57:50
categories:
tags:
---

## 创建用户及组

创建用户组 `groupadd` ，创建用户 `useradd` , 设置密码 `passwd` ，配置sudo权限 `vim /etc/sudoers` 并添加 `userName ALL=(ALL) ALL` , 删除用户 `userdel [-r] userName` , 删除用户组 `groupdel` 

> 删除用户时携带-r表示完全删除，不添加则home及相关数据保留仅删除用户

## 查看用户及组

用户列表文件 `/etc/passwd` , 用户组列表文件 `/etc/group` 


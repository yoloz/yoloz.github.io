---
title: unix\linux常用命令
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

## shell、exec、source执行脚本的区别

### sh方式

使用 `$ sh script.sh` 执行脚本时，当前shell是父进程，生成一个子shell进程，在子shell中执行脚本。脚本执行完毕，退出子shell，回到当前shell.

> `$ ./script.sh` 与 `$ sh script.sh` 等效
> 通过sh执行脚本时，修改的上下文不会影响当前shell

### source方式

使用 `$ source script.sh` 方式，在当前上下文中执行脚本，不会生成新的进程。脚本执行完毕，回到当前shell.

> `source` 方式也叫点命令， `$ . script.sh` 与 `$ source script.sh` 等效。注意在点命令中， `.` 与 `script.sh` 之间有一个空格。
> 通过source执行脚本时，修改的上下文会影响当前shell

### exec方式

使用 `exec command` 方式，会用 `command` 进程替换当前shell进程，并且保持 `PID` 不变。执行完毕，直接退出，不回到之前的shell环境。

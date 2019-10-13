---
title: ps
comments: false
toc: false
date: 2018-11-30 11:11:05
categories: 
tags:
---
查看它的man手册可以看到，ps命令能够给出当前系统中进程的快照。它能捕获系统在某一事件的进程状态。如果你想不断更新查看的这个状态，可以使用top命令。  

ps命令支持三种使用的语法格式

* UNIX 风格，选项可以组合在一起，并且选项前必须有“-”连字符
* BSD 风格，选项可以组合在一起，但是选项前不能有“-”连字符
* GNU 风格的长选项，选项前有两个“-”连字符  

我们能够混用这几种风格，但是可能会发生冲突。

## 显示所有进程

``` bash
 ps -ax #-a代表all,同时加上x参数会显示没有控制终端的进程
 ps -ax | less  #less对文件或其它输出进行分页显示
```

## 通过用户过滤

``` bash
 ps -u 'user'
```

## 通过cpu和内存使用过滤

``` bash
 ps -aux --sort -pcpu | less #CPU使用来降序排序，pcpu是%cpu的别名
 ps -aux --sort -%cpu | less #同上面一样
 ps -aux --sort -pmem | less #内存使用来降序排序，pmem是%mem的别名
 ps -aux --sort -pcpu,+pmem | head -n 10 #cpu降序，内存升序前10条
```

> --sort spec  

Specify sorting order. Sorting syntax is [+|-]key[, [+|-]key[, ...]]. Choose a multi-letter key from the STANDARD FORMAT SPECIFIERS section.
The "+" is optional since default direction is increasing numerical or lexicographic order.  Identical to k.  For example: ps jax --sort=
uid, -ppid, +pid  

``` bash
ps aux | sort -nr -k 4 | head -n 10 | awk '{total+=$4}END{print "sum="total"%"}' #占用最多内存的10个进程的内存占用总和
```

> sort - sort lines of text files  

-n, --numeric-sort compare according to string numerical value
-r, --reverse reverse the result of comparisons 反转(从大到小)
-k, --key=KEYDEF sort via a key; KEYDEF gives location and type 排序key的位置

## 通过进程名过滤

``` bash
 ps -C 'cmd'
 ps -f -C 'cmd'
```

> -C cmdlist  

Select by command name. This selects the processes whose executable name is given in cmdlist.  
-f Do full-format listing.  
-F Extra full format. See the -f option, which -F implies.

## 显示特定进程的线程

``` bash
 ps -Lf 'pid'
```

> -L Show threads, possibly with LWP and NLWP columns.

## 定时刷新信息

ps显示系统当前的进程状态，但是这个结果是静态的, 可以将ps命令和watch命令结合起来实现定时刷新

``` bash
 watch -n 1 'ps -aux --sort -pmem,-pcpu'
 watch -n 1 'ps -aux --sort -pmem,-pcpu | head 10'
```

> -n, --interval seconds to wait between updates

## 树形展示

``` bash
 pstree -aps 'pid'
 ps -jL 'pid' # -j Jobs format(以作业的方式显示进程).
```

> pstree - display a tree of processes.  

-a Show command line arguments.  
-p Show PIDs.  
-s Show parent processes of the specified process.  

ps命令在不同的 UNIX、BSD、Linux 等系统中的参数不尽相同, 不要忘了通过man ps来查看参数

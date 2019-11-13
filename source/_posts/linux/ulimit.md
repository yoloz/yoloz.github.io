---
title: ulimit
comments: false
toc: false
date: 2019-11-13 15:29:46
categories:
tags:
---

参考ulimit的帮助文档（注意：不是man ulimit，而是help ulimit，前者提供的是C语言的ulimit帮助）：

> Modify shell resource limits. Provides control over the resources available to the shell and processes it creates, on systems that allow such control.

可以看出，ulimit提供了对shell（或shell创建的进程）可用资源的管理。除了打开文件数之外，可管理的资源有： 最大写入文件大小、最大堆栈大小、core dump文件大小、cpu时间限制、最大虚拟内存大小等等，help ulimit会列出每个option限制的资源。

``` sh
$ ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) 100
pending signals                 (-i) 15520
max locked memory       (kbytes, -l) 16384
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 15520
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

ulimit -a # 查看所有soft值
ulimit -Ha # 查看所有hard值
ulimit -Hn # 查看nofile的hard值
ulimit -Sn 1000 # 将nofile的soft值设置为1000
ulimit -n 1000 # 同时将nofiles的hard和soft值设置为1000
```

> `ulimit` 无参数相当于 `ulimit -f -S` 指可写入的文件最大size

## soft and hard limit

ulimit对资源的限制区分为soft和hard两类，即同一个资源（如nofile）存在soft和hard两个值。在命令上，ulimit通过-S和-H来区分soft和hard。如果没有指定-S或-H，在显示时指的是soft，而在设置的时候指的是同时设置soft和hard值。

> * 无论何时，soft总是小于等于hard;
> * 无论是超过了soft还是hard，操作都会被拒绝。结合soft<=hard这句话等价于：超过了soft限制，操作会被拒绝。  
> * 一个process可以修改当前process的soft或hard。但修改需满足规则：  
> 1, 修改后soft不能超过hard, 也就是说soft增大时，不能超过hard, 若hard降低到比当前soft还小，那么soft也会随之降低;  
> 2, 非root或root进程都可以将soft在[0-hard]的范围内任意增加或降低;  
> 3, 非root进程可以降低hard，但不能增加hard。即nofile原来是1000，修改为了900，再修改为1000是不行的。（这是一个单向的，又去无回的操作）；  
> 4, root进程可以任意修改hard值
> * JDK的实现中会直接将nofile的soft先改成了和hard一样的值, 即java进程soft和hard一致

## ulimit的修改与生效

关于ulimit的生效，抓住几点即可：

* ulimit的值总是继承父进程的设置。
* ulimit命令可修改当前shell进程的设置，要保证下次生效则修改的地方要具有持久性（至少相当于目标进程而言），如.bashrc或进程的启动脚本，运行中的进程，不受ulimit的修改影响。
* 增加hard值，只能通过root完成。

下面给出两个案例：

**案例1：某非root进程要求2048的nofile，经查看当前soft为1024，hard为4096**  
可以直接在该进程启动脚本中，增加ulimit -nS 2048即可

**案例2：某非root进程要求10240的nofile，经查看当前soft为1024，hard为4096**  
显然，非root用户没法修改只能通过root修改，一般修改/etc/security/limits.conf文件:

``` vim

*  hard nofile 10240
*  soft nofile 10240

```

> 一条记录包含4️列，分别是范围domain（即生效的范围，可以是用户名、group名或*代表所有非root用户）；类型type：即soft、hard，或者-代表同时设置soft和hard；项目item，即ulimit中的资源控制项目，名字枚举可以参考文件中的注释；最后就是value

## 运行中进程的limits的查看

修改ulimit前就启动的进程，如何知道其ulimit值呢, 可以通过查看进程目录下的limits文件

``` sh
$ cat /proc/4660/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             15520                15520                processes
Max open files            2000                 2000                 files
Max locked memory         16777216             16777216             bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       15520                15520                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
```

## 查看系统打开文件数

losf命令虽然作用是”list open files”，但用 `lsof | wc -l` 统计打开文件数不准确, 主要原因有:

* 某些情况下，一行可能显示的是线程，而不是进程，对于多线程的情况，就会误以为一个文件被重复打开了很多次;
* 子进程会共享file handler, 如果用lsof统计，必须使用精巧的过滤条件;

简单和准确的方式是通过/proc目录查看[/proc/sys/fs/file-nr](https://www.kernel.org/doc/Documentation/sysctl/fs.txt)

> 查看一个进程的打开文件数，直接查看目录/proc/$pid/fd里的文件数
> /proc/sys/fs/file-max这个文件控制了系统内核可以打开的全部文件总数

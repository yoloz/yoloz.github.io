---
title: killall,kill,pkill
comments: false
toc: false
date: 2020-03-11 12:08:15
categories:
tags:
---

## killall

killall\[参数\]\[进程名\]
> -Z 只杀死拥有scontext 的进程  
> -e 要求匹配进程名称  
> -I 忽略小写  
> -g 杀死进程组而不是进程  
> -i 交互模式，杀死进程前先询问用户  
> -l 列出所有的已知信号名称  
> -q 不输出警告信息  
> -s 发送指定的信号  
> -v 报告信号是否成功发送  
> -w 等待进程死亡  
> --help 显示帮助信息  
> --version 显示版本显示  

killall命令用于杀死指定名字的进程（kill processes by name）

我们可以使用kill命令杀死指定进程PID的进程，如果要找到我们需要杀死的进程，我们还需要在之前使用ps等命令再配合grep来查找进程，而killall把这两个过程合二为一，是一个很好用的命令。

``` sh
#杀死所有同名进程
$ killall nginx
$ killall -9 bash
#向进程发送指定信号
killall -TERM nginx
killall -KILL nginx
```

## pkill

pkill 和killall 应用方法差不多，也是直接杀死运行中的程式,pkill后的进程名可以不完整，而killall的进程名必须完整的。
关闭已登录的用户:
``` sh
$ who
user1    pts/0        Jun  7 21:50 (125.120.44.33)
user2    pts/1        Jun  7 21:55 (125.120.44.33)
user3    pts/2        Jun  7 22:05 (125.120.44.33)

$ pkill -kill -t pts/0

```

## kill

kill\[参数\]\[进程id\]
> -l  信号，若果不加信号的编号参数，则使用“-l”参数会列出全部的信号名称  
> -a  当处理当前进程时，不限制命令名和进程号的对应关系  
> -p  指定kill 命令只打印相关进程的进程号，而不发送任何信号  
> -s  指定发送信号  
> -u  指定用户  

kill命令用来终止指定的进程（terminate a process）的运行，是Linux下进程管理的常用命令

通常，终止一个前台进程可以使用Ctrl+C键，但是，对于一个后台进程就须用kill命令来终止，我们需要先使用ps/pidof/pstree/top等工具获取进程PID，然后使用kill命令来杀掉该进程。

kill命令是通过向进程发送指定的信号来结束相应进程的。在默认情况下，采用编号为15的TERM信号。TERM信号将终止所有不能捕获该信号的进程。对于那些可以捕获该信号的进程就要用编号为9的kill信号，强行“杀掉”该进程。

kill命令可以带信号号码选项，也可以不带。如果没有信号号码，kill命令就会发出终止信号(15)，这个信号可以被进程捕获，使得进程在退出之前可以清理并释放资源。也可以用kill向进程发送特定的信号。

例如：`kill -2 123`它的效果等同于在前台运行PID为123的进程时按下Ctrl+C键。但是，普通用户只能使用不带signal参数的kill命令或最多使用-9信号。

> 应注意，信号使进程强行终止，这常会带来一些副作用，如数据丢失或者终端无法恢复到正常状态。
>
>要撤销所有的后台作业，可以输入`kill 0`。因为有些在后台运行的命令会启动多个进程，跟踪并找到所有要杀掉的进程的PID是件很麻烦的事。这时，使用`kill 0`来终止所有由当前shell启动的进程，是个有效的方法。

``` sh
#列出所有信号名称
[root@localhost test6]# kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL
 5) SIGTRAP      6) SIGABRT      7) SIGBUS       8) SIGFPE
 9) SIGKILL     10) SIGUSR1     11) SIGSEGV     12) SIGUSR2
13) SIGPIPE     14) SIGALRM     15) SIGTERM     16) SIGSTKFLT
17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU
25) SIGXFSZ     26) SIGVTALRM   27) SIGPROF     28) SIGWINCH
29) SIGIO       30) SIGPWR      31) SIGSYS      34) SIGRTMIN
35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3  38) SIGRTMIN+4
39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12
47) SIGRTMIN+13 48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14
51) SIGRTMAX-13 52) SIGRTMAX-12 53) SIGRTMAX-11 54) SIGRTMAX-10
55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7  58) SIGRTMAX-6
59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX

#只有第9种信号(SIGKILL)才可以无条件终止进程，其他信号进程都有权利忽略,下面是常用的信号：
#HUP    1    终端断线
#INT     2    中断（同 Ctrl + C）
#QUIT    3    退出（同 Ctrl + \）
#TERM   15    终止
#KILL    9    强制终止
#CONT   18    继续（与STOP相反， fg/bg命令）
#STOP    19    暂停（同 Ctrl + Z）


#得到指定信号的数值
[root@localhost test6]# kill -l KILL
[root@localhost test6]# kill -l SIGKILL
[root@localhost test6]# kill -l TERM
[root@localhost test6]# kill -l SIGTERM
[root@localhost test6]#

#先用ps查找进程，然后用kill杀掉
[root@localhost test6]# ps -ef|grep vim 
root      3268  2884  0 16:21 pts/1    00:00:00 vim install.log
root      3370  2822  0 16:21 pts/0    00:00:00 grep vim
[root@localhost test6]# kill 3268 

#彻底杀死进程
命令：kill –9 3268   #-9 强制杀掉进程
```

## kill 进程的几种用法

把ps的查询结果通过管道给grep查找包含特定字符串的进程。管道符“|”用来隔开两个命令，管道符左边命令的输出会作为管道符右边命令的输入`ps -ef|grep pname, kill -9 pid`

### 使用pgrep
pgrep的p表明了这个命令是专门用于进程查询的grep。`pgrep pname, kill  -9 pid`

### 使用pidof
*没错pid of xx，字面翻译过来就是 xx的PID*
`pidof pname, kill  -9 pid` 
和pgrep相比稍显不足的是，pidof必须给出进程的全名

### 使用pkill
*看到pkill想到了什么？没错pgrep和kill！pkill＝pgrep+kill*
`$pkill -９ pname`

### 使用killall
killall和pkill是相似的,不过如果给出的进程名不完整，killall会报错。pkill或者pgrep只要给出进程名的一部分就可以终止进程。
`$killall -9 pname`

### 改进一
`$ps -ef | grep pname | grep -v grep | cut -c 9-15 | xargs kill -s 9`
说明：
“grep firefox”的输出结果是，所有含有关键字“firefox”的进程。
“grep -v grep”是在列出的进程中去除含有关键字“grep”的进程。
“cut -c 9-15”是截取输入行的第9个字符到第15个字符，而这正好是进程号PID。
“xargs kill -s 9”中的xargs命令是用来把前面命令的输出结果（PID）作为“kill -s 9”命令的参数，并执行该命令。
“kill -s 9”会强行杀掉指定进程。

### 改进二
`ps -ef | grep pname | awk '{print $2}' | xargs kill -9` 或者`
kill -s 9  $(ps -aux | grep pname | awk '{print $2}' )`

### 改进三
使用pgrep或pidof缩短命令长度
`$ pgrep pname | xargs kill -s 9`
`kill -s 9 $(pgrep pname)`




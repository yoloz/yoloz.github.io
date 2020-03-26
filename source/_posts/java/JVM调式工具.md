---
title: JVM调试工具
comments: false
toc: false
date: 2020-03-10 11:29:53
categories: java
tags:
---

## pid

在linux环境下，可以通过top命令查看各个进程的cpu使用情况，找出占用较高的pid

## tid

找出pid后通过`top -Hp pid`查看该进程下各个线程的cpu使用情况。任取一个占用cpu较高的线程，将线程id转为16进制`printf “%x\n” tid`

## jstack

通过jstack命令查看CPU利用率高的线程正在做什么`jstack pid |grep tid `

jstack命令生成的thread dump信息包含了JVM中所有存活的线程,大多数情况下会基于thead dump分析当前各个线程的运行情况，如是否存在死锁、是否存在一个线程长时间持有锁不放等等。在dump中，线程一般存在如下几种状态：
* RUNNABLE，线程处于执行中
* BLOCKED，线程被阻塞
* WAITING，线程正在等待

更多信息参考[《javaThreadDump日志分析》](/2020/03/10/java/javaThreadDump日志分析/)

## jmap

使用内存映像工具jmap查看堆内存占用情况`jmap -heap pid`,查看堆的占用情况`jmap -histo pid |less`

用jmap把进程内存使用情况dump到文件中，再用jhat分析查看

``` sh
jmap -dump:format=b,file=dumpFileName pid`
jhat -port 8888 /home/dump.dat
```

> dump出来的文件还可以用MAT、VisualVM等工具查看
>
>如果文件太大，可能需要加上-J-Xmx512m参数以指定最大堆内存，即`jhat -J-Xmx512m -port 8888 /home/dump.dat`然后就可以在浏览器中输入主机地址:8888查看了

## jstat

查看各个区内存和GC的情况

`jstat [ generalOption | outputOptions vmid [interval[s|ms] [count]] ]`

vmid是Java虚拟机ID，在Linux/Unix系统上一般就是进程ID。interval是采样时间间隔。count是采样数目。
`jstat -gc 2860 250 6`采样时间间隔为250ms，采样数为6

> 堆内存 = 年轻代 + 年老代 + 永久代  
> 年轻代 = Eden区 + 两个Survivor区（From和To）
>
>jstat各列含义:  
>S0C、S1C、S0U、S1U： S0:Survivor 0 C:容量（Capacity） U:使用量（Used）  
>EC、EU：Eden区容量和使用量  
>OC、OU：年老代容量和使用量  
>PC、PU：永久代容量和使用量  
>YGC、YGT：年轻代GC次数和GC耗时  
>FGC、FGCT：Full GC次数和Full GC耗时  
>GCT：GC总耗时    

## hprof(Heap/CPU Profiling Tool)

J2SE中提供了一个简单的命令行工具来对java程序的cpu和heap进行 profiling，叫做HPROF。

HPROF实际上是JVM中的一个native的库，它会在JVM启动的时候通过命令行参数来动态加载，并成为JVM进程的一部分。

若要在java进程启动的时候使用HPROF，用户可以通过各种命令行参数类型来使用HPROF对java进程的heap或者 （和）cpu进行profiling的功能。

HPROF产生的profiling数据可以是二进制的，也可以是文本格式的。这些日志可以用来跟踪和分析 java进程的性能问题和瓶颈，解决内存使用上不优的地方或者程序实现上的不优之处。二进制格式的日志还可以被JVM中的HAT工具来进行浏览和分析，用 以观察java进程的heap中各种类型和数据的情况。

在J2SE 5.0以后的版本中，HPROF已经被并入到一个叫做Java Virtual Machine Tool Interface（JVM TI）中。




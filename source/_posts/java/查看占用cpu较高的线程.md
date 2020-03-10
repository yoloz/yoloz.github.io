---
title: 查看占用cpu较高的线程
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



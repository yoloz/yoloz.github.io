---
title: du,df
comments: false
toc: false
date: 2020-02-26 18:17:56
categories: linux
tags:
---

## du

disk usage,是通过搜索文件来计算每个文件的大小然后累加，du能看到的文件只是一些当前存在的，没有被删除的。他计算的大小就是当前他认为存在的所有文件大小的累加和。

``` sh
du -sh  #查看当前目录总共占的容量，不单独列出各子项占用的容量
du -sh file  #查看文件file的大小
du -ah --max-depth=1  #查看当前目录下一级子文件和子目录占用的磁盘容量
```
> a表示显示目录下所有的文件和文件夹（不含子目录）
> h表示以以K，M，G为单位，提高信息的可读性
> max-depth表示目录的深度。 


* 按照空间大小排序
`du|sort -nr|more`

> sort:
>
>-n, --numeric-sort          compare according to string numerical value
>
>-h, --human-numeric-sort    compare human readable numbers (e.g., 2K 1G)

## df

disk free，通过文件系统来快速获取空间大小的信息，当我们删除一个文件的时候，这个文件不是马上就在文件系统当中消失了，而是暂时消失了，当所有程序都不用时，才会根据OS的规则释放掉已经删除的文件， df记录的是通过文件系统获取到的文件的大小，他比du强的地方就是能够看到已经删除的文件，而且计算大小的时候，把这一部分的空间也加上了，更精确了。

``` sh
df #显示磁盘使用情况
df -T #列出文件系统的类型
df -h #以更易读的方式显示目前磁盘空间和使用情况
```

**当文件系统也确定删除了该文件后，这时候du与df就一致了**



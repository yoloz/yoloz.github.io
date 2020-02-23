---
title: java进程占用virt虚拟内存高的问题研究
comments: false
toc: false
date: 2018-12-14 16:52:25
categories: java
tags:
---

## 现象

最近发现线上机器 java 8 进程的 VIRT 虚拟内存使用达到了 50G+，如下图所示：  

![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/java_virt1.png)

## 不管用的 -Xmx

使用 java 的 -Xmx 去限制堆的使用, 没有什么效果。

## 什么是 VIRT

现代操作系统里面分配虚拟地址空间操作不同于分配物理内存。在64位操作系统上，可用的最大虚拟地址空间有16EB，即大概180亿GB。那么在一台只有16G的物理内存的机器上，我也能要求获得4TB的地址空间以备将来使用。例如：

``` c
void *mem = mmap(0, 4ul * 1024ul * 1024ul * 1024ul * 1024ul,PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS | MAP_NORESERVE,-1, 0);
```

当使用 mmap 并设置 MAP_NORESERVE 标志时，并不会要求实际的物理内存和swap空间存在。所以上述代码可以在top中看到使用了 4096g 的 VIRT 虚拟内存，这当然是不可能的，它只是表示使用了 4096GB 的地址空间而已。

## 为什么会用这么多地址空间

那 Java 程序为什么会使用这么多的地址空间呢？使用“pmap -x pid”来查看一下:

``` sh
Address           Kbytes     RSS   Dirty Mode  Mapping
…
00007ff638021000   65404       0       0 -----    [ anon ]
00007ff63c000000     132      36      36 rw---    [ anon ]
00007ff63c021000   65404       0       0 -----    [ anon ]
00007ff640000000     132      28      28 rw---    [ anon ]
00007ff640021000   65404       0       0 -----    [ anon ]
00007ff644000000     132       8       8 rw---    [ anon ]
00007ff644021000   65404       0       0 -----    [ anon ]
00007ff648000000     184     184     184 rw---    [ anon ]
00007ff64802e000   65352       0       0 -----    [ anon ]
00007ff64c000000     132     100     100 rw---    [ anon ]
00007ff64c021000   65404       0       0 -----    [ anon ]
00007ff650000000     132      56      56 rw---    [ anon ]
00007ff650021000   65404       0       0 -----    [ anon ]
00007ff654000000     132      16      16 rw---    [ anon ]
00007ff654021000   65404       0       0 -----    [ anon ]
…
```

发现有很多奇怪的64MB的内存映射，查资料发现这是 glibc 在版本 2.10 引入的 arena 新功能导致。CentOS 6/7 的 glibc 大都是 2.12/ 2.17 了，所以都会有这个问题。这个功能对每个线程都分配一个分配一个本地arena来加速多线程的执行。

> These memory pools are called arenas and the implementation is in arena.c. The first important macro is HEAP_MAX_SIZE which is the maximum size of an arena and it is basically 1MB on 32-bit and 64MB on 64-bit:

HEAP_MAX_SIZE = (2 * DEFAULT_MMAP_THRESHOLD_MAX)  
32-bit [DEFAULT_MMAP_THRESHOLD_MAX = (512 * 1024)] = 1, 048, 576 (1MB)  
64-bit [DEFAULT_MMAP_THRESHOLD_MAX = (4 *1024* 1024 * sizeof(long))] = 67, 108, 864 (64MB)  
32-bit应用程序Arena的大小最大为1MB，64-bit应用程序最大为64MB

在 glibc 的 arena.c 中使用的 mmap() 调用就和之前的示例代码类似：

``` c
p2 = (char *)mmap(aligned_heap_area, HEAP_MAX_SIZE, PROT_NONE,MAP_NORESERVE | MAP_ANONYMOUS | MAP_PRIVATE, -1, 0)
```

之后，只有很小的一部分地址被映射到了物理内存中： `mprotect(p2, size, PROT_READ | PROT_WRITE)` 
因此在一个多线程程序中，会有相当多的 64MB 的 arena 被分配。这个可以用环境变量 MALLOC_ARENA_MAX 来控制。在64位系统中的默认值为 128。

## Java 的特殊性

Java 程序由于自己维护堆的使用，导致调用 glibc 去管理内存的次数较少。更糟的是 Java 8 开始使用 metaspace 原空间取代永久代，而元空间是存放在操作系统本地内存中，那线程一多，每个线程都要使用一点元空间，每个线程都分配一个 arena，每个都64MB，就会导致巨大的虚拟地址被分配。

## 解决办法

Arena内存池主要是用来提高glibc内存分配性能的，

* 限制Arena内存池的个数:

根据Hadoop、Redis等产品的最佳实践建议，尝试设置MALLOC_ARENA_MAX环境变量值为4 `export MALLOC_ARENA_MAX=4` 。  
glibc 2.12版本有几个Arena内存管理的Bug，可能导致参数设置不生效或生效后内存继续往上涨：

> Bug 799327 - MALLOC_ARENA_MAX=1 does not work in RHEL 6.2(glibc 2.12)  

Bug 20425 - unbalanced and poor utilization of memory in glibc arenas may cause memory bloat and subsequent OOM  
Bug 11261 - malloc uses excessive memory for multi-threaded applications  

* 使用Google的tcmalloc替代操作系统自带的glibc管理内存:

将MALLOC_ARENA_MAX设置较小会影响Arena内存池管理上的一些性能，要继续使用MALLOC_ARENA_MAX参数，就需要升级glibc的版本，升级完还不确定高版本的glibc与其他包兼容性上有什么影响，可以使用Google的tcmalloc替代操作系统自带的glibc管理内存。

## 总结

从glibc 2.11（为应用系统在多核心CPU和多Sockets环境中高伸缩性提供了一个动态内存分配的特性增强）版本开始引入了per thread arena内存池，Native Heap区被打散为sub-pools ，这部分内存池叫做Arena内存池。也就是说，以前只有一个main arena，目前是一个main arena（还是位于Native Heap区） + 多个per thread arena，多个线程之间不再共用一个arena内存区域了，保证每个线程都有一个堆，这样避免内存分配时需要额外的锁来降低性能。main arena主要通过brk/sbrk系统调用去管理，per thread arena主要通过mmap系统调用去分配和管理。  
我们来看下线程申请per thread arena内存池的流程：  
Unlimited MALLOC_ARENAS_MAX

``` bash
Thread asks for an per thread arena

Thread gets an per thread arena

Thread fills arena, never frees memory

Thread asks for an new per thread arena .- ...........

When no more per thread arena will be created, reused_arena function will be called to reuse arena already existed.
```

一个Java虚拟机进程究竟能创建多少个arena、每个arena的大小又是多少？
每个arena大小多少参加上文引用部分。Arena数量最大值的计算公式： `maximum number of arenas = NUMBER_OF_CPU_CORES * (sizeof(long) == 4 ? 2 : 8)` 
*一个32位的应用程序进程，最大可创建 2 * CPU总核数个arena内存池（MALLOC_ARENA_MAX），每个arena内存池大小为1MB*  
*一个64位的应用程序进程，最大可创建 8 * CPU总核数个arena内存池（MALLOC_ARENA_MAX），每个arena内存池大小为64MB*  

> 理论归理论，glibc 2.12版本（也就是RHEL 6.x中默认自带的）在arena内存分配和管理上，由于不少的Bug或目前我还没完全弄明白的理论的存在，实际上用pmap看到的1MB或64MB的anonymous memory（缩写为anon）并不完全遵循MALLOC_ARENA_MAX个数设置。

除上述几处bug文章外，还有两个地方的文档显示，MALLOC_ARENA_MAX个数并不是按照设计那样的工作。

如果不考虑内存分配的性能，可使用 `export MALLOC_ARENA_MAX=1` 禁用per thread arena，只用main arena，多个线程共用一个arena内存池。如果考虑到性能，可使用tcmalloc或jemalloc替代操作系统自带的glibc管理内存。

参考
[当Java虚拟机遇上Linux Arena内存池](https://blog.csdn.net/qq_36510261/article/details/78392409)  
[Java 进程占用 VIRT 虚拟内存超高的问题研究](https://www.cnblogs.com/seasonsluo/p/java_virt.html)
[内存优化总结:ptmalloc、tcmalloc和jemalloc](http://www.cnhalo.net/2016/06/13/memory-optimize/)


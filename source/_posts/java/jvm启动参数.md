---
title: jvm启动参数
comments: false
toc: false
date: 2019-02-15 13:31:51
categories: java
tags:
---

java启动参数共分为三类

* 其一是标准参数（-），所有的JVM实现都必须实现这些参数的功能，而且向后兼容；
* 其二是非标准参数（-X），默认jvm实现这些参数的功能，但是并不保证所有jvm实现都满足，且不保证向后兼容；
* 其三是非Stable参数（-XX），此类参数各个jvm实现会有所不同，将来可能会随时取消，需要慎重使用；
> 非标准参数又称为扩展参数

## verbose 

* -verbose:class 

 输出jvm载入类的相关信息，当jvm报告说找不到类或者类冲突时可此进行诊断。

* -verbose:gc 

 输出每次GC的相关情况。

* -verbose:jni 

 输出native方法调用的相关情况，一般用于诊断jni调用错误信息。

## 扩展参数

* -Xms512m

设置JVM促使内存为512m。此值可以设置与-Xmx相同，以避免每次垃圾回收完成后JVM重新分配内存。

* -Xmx512m

设置JVM最大可用内存为512M

* -Xmn200m

设置年轻代大小为200M。整个堆大小=年轻代大小 + 年老代大小 + 持久代大小。持久代一般固定大小为64m，所以增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。

* -Xss128k

设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。在相同物理内存下，减小这个值能生成更多的线程。但是操作系统对一个进程内的线程数还是有限制的，不能无限生成，经验值在3000~5000左右。

* -Xloggc:file

 与-verbose:gc功能类似，只是将每次GC事件的相关情况记录到一个文件中，文件的位置最好在本地，以避免网络的潜在问题。

 > 若与verbose命令同时出现在命令行中，则以-Xloggc为准。

* -Xprof

 跟踪正运行的程序，并将跟踪数据在标准输出输出；适合于开发环境调试。

## 非Stable参数

 用-XX作为前缀的参数列表在jvm中可能是不健壮的，SUN也不推荐使用，后续可能会在没有通知的情况下就直接取消了；但是由于这些参数中的确有很多是对我们很有用的，比如我们经常会见到的-XX:PermSize、-XX:MaxPermSize等等；

### 行为参数列表

 |  参数及其默认值   | 描述  |
|  :----  | :----  |
| -XX:-DisableExplicitGC | 禁止调用System.gc()；但jvm的gc仍然有效 |
| -XX:+MaxFDLimit | 最大化文件描述符的数量限制 |
| -XX:+ScavengeBeforeFullGC | 新生代GC优先于Full GC执行 |
| -XX:+UseGCOverheadLimit | 在抛出OOM之前限制jvm耗费在GC上的时间比例 |
| **-XX:-UseConcMarkSweepGC** | **对老生代采用并发标记交换算法进行GC** |
| **-XX:-UseParallelGC** | **启用并行GC** |
| -XX:-UseParallelOldGC | 对Full GC启用并行，当-XX:-UseParallelGC启用时该项自动启用 |
| **-XX:-UseSerialGC** | **启用串行GC** |
| -XX:+UseThreadPriorities | 启用本地线程优先级 |

> 上面表格中黑体的三个参数代表着jvm中GC执行的三种方式，即串行、并行、并发；
>
> 串行（SerialGC）是jvm的默认GC方式，一般适用于小型应用和单处理器，算法比较简单，GC效率也较高，但可能会给应用带来停顿；
>
>并行（ParallelGC）是指GC运行时，对应用程序运行没有影响，GC和app两者的线程在并发执行，这样可以最大限度不影响app的运行；
>
>并发（ConcMarkSweepGC）是指多个线程并发执行GC，一般适用于多处理器系统中，可以提高GC的效率，但算法复杂，系统消耗较大；

### 性能调优参数列表

 |  参数及其默认值   | 描述  |
|  :----  | :----  |
| -XX:LargePageSizeInBytes=4m | 设置用于Java堆的大页面尺寸 |
| -XX:MaxHeapFreeRatio=70 | GC后java堆中空闲量占的最大比例 |
| **-XX:MaxNewSize=size** | **新生成对象能占用内存的最大值** |
| **-XX:MaxPermSize=64m** | **老生代对象能占用内存的最大值** |
| -XX:MinHeapFreeRatio=40 | GC后java堆中空闲量占的最小比例 |
| -XX:NewRatio=2 | 新生代内存容量与老生代内存容量的比例 |
| **-XX:NewSize=2.125m** | **新生代对象生成时占用内存的默认值** |
| -XX:ReservedCodeCacheSize=32m | 保留代码占用的内存容量 |
| -XX:ThreadStackSize=512 | 设置线程栈大小，若为0则使用系统默认值 |
| -XX:+UseLargePages | 使用大页面内存 |

> 我们在日常性能调优中基本上都会用到以上黑体的这几个属性

## 调试参数列表

 |  参数及其默认值   | 描述  |
|  :----  | :----  |
| -XX:-CITime | 打印消耗在JIT编译的时间 |
| -XX:ErrorFile=./hs_err_pid\<pid\>.log | 保存错误日志或者数据到文件中 |
| -XX:-ExtendedDTraceProbes | 开启solaris特有的dtrace探针 |
| **-XX:HeapDumpPath=./java_pid\<pid\>.hprof** | **指定导出堆信息时的路径或文件名** |
| **-XX:-HeapDumpOnOutOfMemoryError** | **当首次遭遇OOM时导出此时堆中相关信息** |
| -XX:OnError="\<cmd args\>;\<cmd args\>" | 出现致命ERROR之后运行自定义命令 |
| -XX:OnOutOfMemoryError="\<cmd args\>;\<cmd args\>" | 当首次遭遇OOM时执行自定义命令 |
| -XX:-PrintClassHistogram | 遇到Ctrl-Break后打印类实例的柱状信息，与jmap -histo功能相同 |
| **-XX:-PrintConcurrentLocks** | **遇到Ctrl-Break后打印并发锁的相关信息，与jstack -l功能相同** |
| -XX:-PrintCommandLineFlags | 打印在命令行中出现过的标记 |
| -XX:-PrintCompilation | 当一个方法被编译时打印相关信息 |
| -XX:-PrintGC | 每次GC时打印相关信息 |
| -XX:-PrintGC Details | 每次GC时打印详细信息 |
| -XX:-PrintGCTimeStamps | 打印每次GC的时间戳 |
| -XX:-TraceClassLoading | 跟踪类的加载信息 |
| -XX:-TraceClassLoadingPreorder | 跟踪被引用到的所有类的加载信息 |
| -XX:-TraceClassResolution | 跟踪常量池 |
| -XX:-TraceClassUnloading | 跟踪类的卸载信息 |
| -XX:-TraceLoaderConstraints | 跟踪类加载器约束的相关信息 |

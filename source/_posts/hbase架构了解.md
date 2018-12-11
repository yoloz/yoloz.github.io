---
title: hbase架构了解
comments: false
toc: true
date: 2018-12-03 12:06:58
categories: hbase
tags:
---
*内容和结构主要来自[Carol McDonald的文章](https://mapr.com/blog/in-depth-look-hbase-architecture/#.VdMxvWSqqko)，对照官方文档梳理学习了一遍，记录备忘。*
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch1.jpg)  
它们的主要功能有：  
<!--more-->
**Hmaster:**  
* 管理HregionServer，实现负载均衡；
* 管理和分配Hregion,如Hregion split时分配新的Hregion,HregionServer退出时迁移其下的Hregion到其他HregionServer上；
* DDL操作(table,columnfamily的增删改等)；
* 管理namespace和table的元数据(存储在hdfs上)；
* 权限控制(ACL)；  

**HregionServer:**  
* 存放和管理本地的Hregion;
* 读写hdfs，管理table中的数据；
* client通过HregionServer读写数据；  

**zookeeper:**  
* 存放Hbase集群的元数据及集群的状态信息；
* Hmaster主从节点的active;  

HRegion所处理的数据尽量和数据所在的DataNode在一起，实现数据的本地化。数据本地化并不是总能实现，如在HRegion移动(Split)时，需要等下一次Compact才能继续回到本地化。Hmaster和namenode都支持多个热备份，使用zookeeper来做协调。Hregionserver和datanode一般会放在相同的server上实现数据的本地化。如下：![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch2.png)

## Regions
Hbase使用rowkey将表水平分割成多个hregion,region分配给相应的regionserver管理.如下：![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch3.png)![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch4.png)

### Region
* A table can be divided horizontally into one or more regions. A region contains a contiguous, sorted range of rows between a start key and an end key;
* Each region is 1GB in size (default)；
* A region of a table is served to the client by a RegionServer；
* A region server can serve about 1,000 regions (which may belong to the same table or different tables)；  
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch5.png)

### Region Split
最初一个表一个region，随着数据增多，region分裂成两个，两个新的region会在同一个HRegionServer中创建，它们各自包含父region一半的数据，split完成后，父region下线，而新的两个子region向HMaster注册上线。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch6.png)

### Read Load Balancing
分割起初发生在一个regionserver上，由于负载均衡，master会将新生成的region分配到其他的regionserver上，这样会有可能一些regionserver处理的数据暂时在远程机器上，在major compact的时候会将远程数据文件移动到本地来。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch7.png)

## HBase Hmaster
HMaster没有单点故障问题，可以启动多个HMaster，通过ZooKeeper的Master Election机制保证同时只有一个HMaster出于Active状态，其他的HMaster则处于热备份状态。主要职责有：
* 协调HRegionServer  
* * 启动时region的分配，以及负载均衡和修复时region的重新分配;  
* * 监控集群中所有regionServer的状态(通过Heartbeat和监听ZooKeeper中的状态)。  
* Admin职能，创建、删除、修改Table的定义。如下：
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch8.png)

## ZooKeeper: The Coordinator
Hbase使用zookeeper来协调集群的管理。如下：
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch9.png) 

### How the Components Work Together  
zooKeeper协调集群所有节点的共享信息，在master和regionserver连接到zooKeeper后创建Ephemeral节点，使用心跳机制维持这个节点的存活状态，如果某个Ephemeral节点实效，则HMaster会收到通知，并做相应的处理。如下：![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch10.png)  

## HBase First Read or Write
Hbase中有一个特殊的目录表META表，存储了集群的所有regions位置，zookeeper存储了这个meta表的位置。大概流程如下：  
1, 从zooKeeper中获取存储meta表的regionserver的位置；  
2, 从meta中查询用户table对应请求的rowkey所在的regionserver位置；  
3, 从查询到regionServer中获取数据；  
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch11.png)

## HBase Meta Table
hbase:meta表存储了所有region的位置信息(The -ROOT- table was removed in HBase 0.96.0)，结构如下：
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch12.png)
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch13.png)

## Region Server Components
regionserver一般和hdfs的datanode在同一台机器上运行，实现数据的本地性，包含多个region，由WAL、BlockCache、MemStore、HFile组成。如下：
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch14.png)  
* WAL: Write Ahead Log is a file on the distributed file system. The WAL is used to store new data that hasn't yet been persisted to permanent storage; it is used for recovery in the case of failure.
* BlockCache: is the read cache. It stores frequently read data in memory. Least Recently Used data is evicted when full.
* MemStore: is the write cache. It stores new data which has not yet been written to disk. It is sorted before writing to disk. There is one MemStore per column family per region.
* Hfiles store the rows as sorted KeyValues on disk.

## HBase Write
当客户端发起一个Put请求时，首先它从hbase:meta表中查出该Put数据最终需要去的regionserver。然后客户端将Put请求发送给相应的regionserver，在regionserver中它首先会将该Put操作写入WAL日志文件中(Flush到磁盘中)。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch15.png)  
写完WAL日志文件后，regionserver根据Put中的tablename和rowkey找到对应的region，并根据column family找到对应的store，并将Put写入到该store的memstore中。此时写成功，并返回通知客户端。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch16.png)

## HBase MemStore
MemStore是一个In Memory Sorted Buffer，在每个store中都有一个memstore，store对应region中的columnfamily。如下：
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch17.png)

## HBase Region Flush
当memstore积累满数据后，其中的内容(排序集合)flush进storeFile(hfile)中。注意的是memstore的最小flush单元是region而不是单个memstore(即一个flush，其中所有的都flush)。可能这也是hbase中的columnfamily不能无限制的增加的原因(region中很多columnfamily同时flush)。在flush过程中，还会在尾部追加一些数据，其中就包括flush时最大的WAL sequence值，告诉hbase这个hfile写入的最新数据的序列，在recover时就知道从哪里开始，在HRegion启动时，这个sequence会被读取，并取最大的作为下一次更新时的起始sequence。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch18.png)

## HBase HFile
数据以key/value(cell)的形式顺序的存储在storefile中，在memstore积累到足够的数据后flush到磁盘生成hfile(memstore中存储的cell遵循相同的排列顺序，所以是顺序写，性能很高，不需要不停的移动磁盘指针)。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch19.png)

### HBase HFile Structure
Hfile的格式经历三次更改，现在是v3,详情官方文档(Appendix H: HFile format)。

## HBase Read Merge
如前文所述，每个memstore含有多个hfile,读取时扫描过多文件会影响性能。扫瞄的顺序依次是：BlockCache、MemStore、StoreFile(HFile)。其中StoreFile的扫瞄先会使用Bloom Filter过滤那些不可能符合条件的HFile，然后使用Block Index快速定位Cell，并将其加载到BlockCache中，然后从BlockCache中读取。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch20.png)

## HBase Compaction
compaction分为两种：
### minor compact
选取一些小的、相邻的hfile合并成一个更大的hfile，在这个过程中不会处理已经deleted或expired的cell;
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch21.png)

### major compact
所有的hfile合并成一个hfile，在这个过程中，标记为deleted的cell会被删除，而那些已经expired的cell会被丢弃。major compaction的结果是一个HStore只有一个hfile存在(one hfile per column family)。可以手动或自动触发，由于它会引起很多的IO操作而影响性能，一般建议安排在周末等比较闲的时间。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch22.png)

## HDFS Data Replication
hbase依赖hdfs来确保数据不丢失，hdfs将数据备份到其他节点上。

## HBase Crash Recovery
当zookeeper监测不到regionserver的心跳包，通知master这个regionserver宕机。master就会对宕机server中的region分配给活着的regionserver。
![](https://bp-1252402719.cos.ap-shanghai.myqcloud.com/HBaseArch23.png)

## 参考
[深入HBase架构解析](http://www.blogjava.net/DLevin/archive/2015/08/22/426877.html)
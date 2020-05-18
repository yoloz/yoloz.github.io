---
title: hbase-2.2.4伪分布式
comments: false
toc: false
date: 2020-05-18 17:20:36
categories:
tags:
---

`tar -zxvf hbase-2.2.4-bin.tar.gz -C ~`

## 单机

**Requirements:jdk**

解压后执行`bin/start-hbase.sh`即可，浏览器访问http://ip:16010
常用命令如下:

``` sh
$ bin/hbase shell

#Create a table
hbase(main):001:0> create 'test', 'cf'
0 row(s) in 0.4170 seconds

=> Hbase::Table - test

#List Information About your Table
hbase(main):002:0> list 'test'
TABLE
test
1 row(s) in 0.0180 seconds

=> ["test"]

#use the describe command to see details
hbase(main):003:0> describe 'test'
Table test is ENABLED
test
COLUMN FAMILIES DESCRIPTION
{NAME => 'cf', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE =>
'false', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', MIN_VERSIONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WRITE => 'f
alse', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 'false', COMPRESSION => 'NONE', BLOCKCACHE => 'true', BLOCKSIZE
 => '65536'}
1 row(s)
Took 0.9998 seconds

#Put data into your table
hbase(main):003:0> put 'test', 'row1', 'cf:a', 'value1'
0 row(s) in 0.0850 seconds

hbase(main):004:0> put 'test', 'row2', 'cf:b', 'value2'
0 row(s) in 0.0110 seconds

hbase(main):005:0> put 'test', 'row3', 'cf:c', 'value3'
0 row(s) in 0.0100 seconds

#Scan the table for all data at once
hbase(main):006:0> scan 'test'
ROW                                      COLUMN+CELL
 row1                                    column=cf:a, timestamp=1421762485768, value=value1
 row2                                    column=cf:b, timestamp=1421762491785, value=value2
 row3                                    column=cf:c, timestamp=1421762496210, value=value3
3 row(s) in 0.0230 seconds

#Get a single row of data
hbase(main):007:0> get 'test', 'row1'
COLUMN                                   CELL
 cf:a                                    timestamp=1421762485768, value=value1
1 row(s) in 0.0350 seconds

#Disable/Enable a table
hbase(main):008:0> disable 'test'
0 row(s) in 1.1820 seconds

hbase(main):009:0> enable 'test'
0 row(s) in 0.1770 seconds

#Drop the table
hbase(main):011:0> drop 'test'
0 row(s) in 0.1370 seconds
```

## 伪分布式

**Requirements:jdk, hadoop**

### 修改配置文件

修改配置hbase-site.xml

``` xml
<property>
  <name>hbase.cluster.distributed</name>
  <value>true</value>
</property>
<property>
  <name>hbase.rootdir</name>
  <value>hdfs://${hostname}:9000/hbase</value>
</property>
#remove existing configuration for 'hbase.tmp.dir' and 'hbase.unsafe.stream.capability.enforce'
```

>You do not need to create the directory in HDFS. HBase will do this for you. If you create the directory, HBase will attempt to do a migration, which is not what you want.

### 启动

Use the `bin/start-hbase.sh` command to start HBase. If your system is configured correctly, the jps command should show the HMaster and HRegionServer processes running.

>出现:Error: JAVA_HOME is not set则手动在conf/hbase-env.sh中添加

### Check the HBase directory in HDFS

If everything worked correctly, HBase created its directory in HDFS. In the configuration above, it is stored in /hbase/ on HDFS. You can use the hadoop fs command in Hadoop’s bin/ directory to list this directory.

``` sh
$ bin/hadoop fs -ls /hbase
Found 7 items
drwxr-xr-x   - hbase users          0 2014-06-25 18:58 /hbase/.tmp
drwxr-xr-x   - hbase users          0 2014-06-25 21:49 /hbase/WALs
drwxr-xr-x   - hbase users          0 2014-06-25 18:48 /hbase/corrupt
drwxr-xr-x   - hbase users          0 2014-06-25 18:58 /hbase/data
-rw-r--r--   3 hbase users         42 2014-06-25 18:41 /hbase/hbase.id
-rw-r--r--   3 hbase users          7 2014-06-25 18:41 /hbase/hbase.version
drwxr-xr-x   - hbase users          0 2014-06-25 21:49 /hbase/oldWALs
```
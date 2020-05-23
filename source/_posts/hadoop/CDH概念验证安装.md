---
title: CDH概念验证安装
comments: false
toc: false
date: 2020-05-20 17:42:02
categories:
tags:
---

## Cloudera Manager Server Installer

``` sh
$ wget https://archive.cloudera.com/cm6/6.3.1/cloudera-manager-installer.bin
$ chmod u+x cloudera-manager-installer.bin
$ sudo ./cloudera-manager-installer.bin
```
> 6.3.3及以上requires authentication
> 会安装oracle_jdk，需要网络通畅或者sudo proxychains ./cloudera-manager-installer.bin

## 通过页面继续安装

地址:http://ip:7180 用户密码admin/admin
在提供SSH登录凭据这一步骤中需要sudo无密码，可如下操作:

```sh
sudo groupadd cloudera
sudo useradd -m cloudera -g cloudera
sudo passwd cloudera
sudo vim /etc/sudoers
#添加　cloudera ALL =(ALL) NOPASSWD: ALL
```

然后选择其他用户填写cloudera,下面的密码即刚才passed设置的

**安装失败。无法接收 Agent 发出的检测信号**
排除下面列出的选项后，执行`sudo service cloudera-scm-agent status`可以发现提示如下:

``` log
Traceback (most recent call last):
May 22 07:05:06 cdh_183 cm[11441]:   File "/opt/cloudera/cm-agent/lib/python2.7/site-packages/cmf/main.py", line 105, in main_impl
May 22 07:05:06 cdh_183 cm[11441]:     ag.configure_service()
May 22 07:05:06 cdh_183 cm[11441]:   File "/opt/cloudera/cm-agent/lib/python2.7/site-packages/cmf/agent.py", line 608, in configure_service
May 22 07:05:06 cdh_183 cm[11441]:     raise Exception("Hostname is invalid; it contains an underscore character.")
May 22 07:05:06 cdh_183 cm[11441]: Exception: Hostname is invalid; it contains an underscore character.
```

> The Internet standards (Request for Comments) for protocols mandate that component hostname labels may contain only the ASCII letters 'a' through 'z' (in a case-insensitive manner), the digits '0' through '9', and the hyphen ('-'). The original specification of hostnames in RFC 952, mandated that labels could not start with a digit or with a hyphen, and must not end with a hyphen. However, a subsequent specification (RFC 1123) permitted hostname labels to start with digits. No other symbols, punctuation characters, or white space are permitted.  
> So, having hostnames(cdh_183) with _ is illegal

**集群设置中Hive报错**

数据库设置中选择默认的嵌入式postgresql库，执行到命令详细信息步骤报错如下:

``` log
Run a set of services for the first time
仅完成 2/3 个步骤。首个失败：命令 (Create Hive Metastore database tables (53)) 已失败
依次运行 8 步骤
仅完成 2/3 个步骤。首个失败：命令 (Create Hive Metastore database tables (53)) 已失败	
并行运行 3 步骤
仅完成 2/3 个步骤。首个失败：命令 (Create Hive Metastore database tables (53)) 已失败
正在创建 Hive Metastore 数据库表
命令 (Create Hive Metastore database tables (53)) 已失败
Hive Metastore Server (cdh183)
创建 Hive Metastore 数据库表
Command aborted because of exception: Command timed-out after 150 seconds
Hive Metastore Server (cdh183)
```

缺少驱动包执行`sudo cp CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/postgresql-42.2.5.jar CDH-6.3.2-1.cdh6.3.2.p0.1605554/lib/hive/lib/`后重试即可。

**虚拟机重启后默认cloudera manager会自启，页面访问http://ip:7180则需等待一会**

**Canary 测试无法为 /tmp/.cloudera_health_monitoring_canary_files 创建父目录。**
一般过一会儿就好了，如果没好可以手动创建
执行`sudo -u hdfs hdfs dfs -mkdir /tmp`即可，稍后页面告警就会消失

>sudo -u hdfs hdfs dfs -ls /  
>修改hdfs的根目录的权限： `hdfs dfs -chmod 777 /`  
>报错**chmod: changing permissions of '/': Permission denied. user=root is not the owner of inode=/**  
>默认用户是hdfs，执行`sudo -u hdfs hdfs dfs -chmod 777 /`即可  

**Error applying authorization policy on hive configuration: org.apache.hadoop.security.AccessControlException: Permission denied: user=hive, access=WRITE, inode="/tmp":hdfs:supergroup:drwxr-xr-x**

`sudo -u hdfs hdfs dfs -chmod a+rwx /tmp`这样/tmp除了所有者有读写执行权限外，其他用户也有读写执行权限


**Namenode and SecondaryNamenode Have different heapsizes**

To change the heap size for NameNode and Secondary NameNode:
* open the HDFS service in Cloudera Manager
* select "Configuration" from the menu
* enter "Java Heap Size " in the search field
Here you can configure heap size for NameNode and Secondary NameNode.

---
title: mysql使用
comments: false
toc: false
date: 2020-02-20 14:02:17
categories:
tags:
---

## limit语句

``` sql
select * from "operativeLogs" ORDER BY "sysTimestamp" DESC LIMIT 5 OFFSET 5    #postgresql
select * from "operativeLogs" ORDER BY "sysTimestamp" DESC LIMIT 5,10        #mysql
```

## 用户

``` sql
#创建
mysql>CREATE USER 'pig'@'192.168.1.101' IDENDIFIED BY '123456';
mysql>create user 'pig'@'%' identified by '123456';
#修改密码
mysql>SET PASSWORD FOR 'username'@'host' = PASSWORD('newpassword');
mysql>SET PASSWORD = PASSWORD("newpassword");   #修改当前用户密码
#删除用户
mysql>DROP USER 'username'@'host';
```

## 权限

### 赋权

GRANT privileges ON dbName.tbName TO 'username'@'host'
>* privileges(SELECT,INSERT,UPDATE...),所有权限ALL表示;
>* 所有数据库或所有表可用\*表示
>* 能给其他用户授权则后面要加上WITH GRANT OPTION

``` sql
#赋权
mysql>GRANT SELECT,INSERT ON test.user TO 'pig'@'%'; #test库下user表的查询插入权限
mysql>GRANT ALL ON *.* TO 'pig'@'%'; #所有权限
mysql>GRANT ALL ON *.* TO 'pig'@'%' WITH GRANT OPTION;  #能赋权给其他用户
#查看
mysql>SHOW GRANTS FOR 'pig'@'%';
```

### 撤权

REVOKE privileges ON databasename.tablename FROM 'username'@'host';
> 赋权和撤权的对象要对应，否则撤权会无效，如：
>
>授权的时候是这样的(或类似的):GRANT SELECT ON test.user TO 'pig'@'%', 则使用REVOKE SELECT ON *.* FROM 'pig'@'%';并不能撤销该用户对test数据库中user表的SELECT 操作;
>
>授权的时候是这样的:GRANT SELECT ON *.* TO 'pig'@'%';则REVOKE SELECT ON test.user FROM 'pig'@'%';也不能撤销该用户对test数据库中user表的Select权限.

## 导入导出

### 导入导出查询结果

``` sql
select语句 into outfile '/opt/file';

load data local infile '/opt/file' into table 表名 character set utf8;
```

### 导入导出结构和数据

mysqldump -u用户名 -p [-d|-t] db_name [tbl_name ...]
> -d:do not write any table row information,即不导出表内容；
>
> -t: do not write create table statements that create each dumped tables,即不导出创建表结构；
>
> 如果只有db_name则导出全库，否则只导出库中的指定表；

* 导出

``` sh
mysqldump -uxxx -p -d database > database.sql #导出数据库表结构
mysqldump -uxxx -p database > database.sql #导出数据库表结构和数据
mysqldump -uxxx -p -t database tablename > tablename.sql #导出数据表数据
mysqldump -uxxx -p database tablename > tablename.sql #导出数据表结构和数据
```
* 导入

``` sql
#方式一
$ mysql -u用户名 -p  
mysql> create database test;  
mysql> exit;  
$ mysql -u用户名 -p test < /home/test.sql
#方式二  
$ mysql -u用户名 -p   
mysql> create database test;  
mysql> use test;  
mysql> source /home/test.sql;  
mysql> exit;
```
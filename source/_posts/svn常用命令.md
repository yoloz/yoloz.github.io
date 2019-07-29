---
title: svn常用命令
comments: false
toc: false
date: 2018-12-05 15:19:46
categories: tools
tags:
---

## 检出(checkout)

``` bash
svn co [--revision] http(svn)://路径(目录或文件的全路径) [本地目录全路径] [--username 用户名] [--password 密码]
```

<!--more-->

> * checkout简写co;
> * 初次操作svn时，不带username, password会提示输入用户信息，然后系统会记录下来，以后使用如果没带username则使用原先记录的；命令中带username则使用命令中的用户；
> * 不带--password参数传输密码会提示输入密码，建议不要用明文的--password 选项；
> * username 与 password前是两个短线，不是一个;
> * 不指定本地目录，则检出到当前目录下;
> * --revision参数(检出具体版本)位置也可以放后面，如:

``` bash
svn checkout http://siphon.googlecode.com/svn/trunk/ siphon -r r791  #-r [--revision]
svn checkout -r r791 http://siphon.googlecode.com/svn/trunk/ siphon
```

### 检出不包括源文件夹目录

比如我要checkout trunk/下面的所有文件，但是不包括trunk文件夹，可以在svn文件夹后面打个空格再加个“.”就行了，如
 `svn co http://192.168.1.10/svn/project/trunk/ . /home/DSP-OPEN`

## 更换svn帐号

* 临时更换, 命令中带上--username选项；
* 全局更换 `svn propset --username xxx` ；

## 创建branch

``` bash
svn cp http://example.com/repos/myproject/trunk http://example.com/repos/myproject/branches/xxx_xxx -m 'create branch xxx_xxx'
```

> copy简写cp, -m: 描述信息  

其实SVN并没有Branch的内部概念。我们只是创建了一个repo的副本，并自己赋予这个副本作为Branch的意义，这与git中的Branch有很大不同。  
需要注意的是Branch和Trunk使用同一套版本号，也就是说无论在Branch还是Trunk的提交都会引起主版本号的增加。这是因为svn copy只支持同一个repository内的文件copy，并不支持跨repository的copy，所以新创建的Branch和Trunk都属于同一个repository。

## 合并

假设现在Branch上fix了一系列的bug，现在我们想把针对Branch的改变同步到Trunk上。
1, 保证当前Branch分支是clean的，也就是说使用svn status看不到任何的本地修改。
2, 命令行下切换到Trunk目录中 `cd /xxxx/trunk` ，使用 `svn merge http://example.com/repos/myproject/branches/xxx_xxx` 来将Branch分支上的改动merge回Trunk下。
3, 如果出现merge冲突则进行解决，然后执行 `svn ci -m 'description'` 来提交变动。

> commit简写ci  

### 指定合并版本

指定Branch上那些版本变更可以合并到Trunk中

``` bash
svn merge  http://example.com/repos/myproject/branches/xxx_xxx -r150:HEAD
```

将Branch的从版本150到当前版本的所有改动都合并到Trunk中。  

> 将Trunk中的某些更新合并到Branch中方法同上，切换到Branch目录中 `cd /xxxx/branches/xxx_xxx` , 然后执行 `svn merge http://example.com/repos/myproject/trunk`

### 查看合并情况

使用svn mergeinfo来查看merge情况.
查看当前Branch中已经有那些改动被合并到Trunk中:

``` bash
cd /xxx/trunk
svn mergeinfo http://example.com/repos/myproject/branches/xxx_xxx
```

查看Branch中那些改动还未合并

``` bash
cd /xxx/trunk
svn merginfo http://example.com/repos/myproject/branches/xxx_xxx --show-revs eligible
```

> Trunk中的更新合并到Branch情况查看同上。

## 更换地址

``` bash
svn sw --relocate old_addr new_addr
```

> switch简写sw  

查看svn信息 `svn info`

## 忽略文件

``` bash
svn ps svn:ignore *.class .
svn ps svn:ignore -R -F .svnignore .
```

> propset PROPNAME PROPVAL PATH...  

svn:ignore A list of file glob patterns to ignore, one per line.  
-F [--file] ARG: read property value from file ARG  
-R [--recursive] : descend recursively, same as --depth=infinity

.svnignore文件中定义多个忽略文件及目录

``` java
*.class
.idea
*.impl
```

## svn revert

恢复整个目录 `svn revert Dir .`

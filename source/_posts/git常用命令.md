---
title: git常用命令
comments: false
toc: false
date: 2018-12-10 16:25:42
categories: tools
tags:
---

获取一个url对应的远程git repo,创建一个本地copy`git clone url`
clone指定分支`git clone -b 分支名 url`,如：git clone -b v2.8.1 https://xxx.git  

<!--more-->

## commit相关
git有一个暂存区(staging area),可以放入新添加的文件或加入新的改动。`commit`是将暂存区的代码提交到本地仓库，不是我们disk上的改动(disk可见的是工作区)。`git add .`会递归地将工作区的改动文件添加到本地的暂存区，两者可以合并成一个命令`git commit -am "something"`，提交到远程服务端`git push`。
> 如果有新增文件，则`git add .`和`git commit -m "something"`不可合并；  
`git commit -am "something"`只能提交已跟踪过的改动文件,新文件是untracking file；

## diff相关
`git diff`工作目录中的文件和暂存区快照之间的差异，即修改后还没暂存起来(commit)的变化内容；
`git diff --cached`已经暂存起来的文件和上次提交时的快照之间的差异；
`git diff HEAD`工作目录中的文件和上次提交之间的改动；
`git diff [version tag]`查看指定版本之后的改动；

## branch相关
查看本地分支`git branch`  
查看远程分支`git branch -r`  
查看所有分支`git branch -a`
本地创建新分支`git branch xxxx`
切换到分支xx`git checkout xx`
创建分支的同时切换到该分支上`git checkout -b xxx`
将新分支推送到远程repo上`git push origin xxx`
删除本地分支`git branch -d xxx`
删除远程分支`git push origin --delete xxx`  

有时候我们需要创建一个干净的分支，其不继承任何提交没有父节点，而上文的`git checkout xx`创建的分支xx是有父节点的，包含了历史提交。流程如下：
1,创建干净的分支`git checkout --orphan xx`；  
2,删除工作目录中其他分支存在的内容`git rm -rf .`;  
3,给分支xx添加内容`git add file1 file2...`;  
4,提交到本地仓库`git commit -m "something"`;  
5,推送到远程仓库`git push origin xx`;

## 撤销相关
git撤销操作主要有如下方式:
撤销上一次提交，并将暂存区文件重新提交`git commit --amend`;
拉取暂存区文件替换工作区文件`git checkout file1 dir1...`;
拉取版本库文件到暂存区,不影响工作区`git reset HEAD file1 dir1...`;

> 第一种情况下--amend会打开编辑文件其中快捷键^表示ctrl,M表示alt(linux)  
meta键是以前MIT计算机键盘上的的一个特殊键

## 更换地址
查看远程仓库信息`git remote -v`
本地仓库更换远程仓库地址
```
git remote rm origin
git remote add origin new_addr
```


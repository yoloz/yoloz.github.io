---
title: git
comments: false
toc: false
date: 2018-12-10 16:25:42
categories: other
tags:
---

## clone相关

获取一个url对应的远程git repo, 创建一个本地copy `git clone url`  
**clone指定分支** `git clone -b 分支名 url` , 如：`git clone -b v2.8.1 https://xxx.git`

## commit相关

git有一个暂存区(staging area), 可以放入新添加的文件或加入新的改动。 `commit` 是将暂存区的代码提交到本地仓库，不是我们disk上的改动(disk可见的是工作区)。 `git add .` 会递归地将工作区的改动文件添加到本地的暂存区，两者可以合并成一个命令 `git commit -am "something"` ，提交到远程服务端 `git push` 。

> 如果有新增文件，则 `git add .` 和 `git commit -m "something"` 不可合并；  

`git commit -am "something"` 只能提交已跟踪过的改动文件, 新文件是untracking file；

## diff相关

`git diff` 工作目录中的文件和暂存区快照之间的差异，即修改后还没暂存起来(commit)的变化内容；  
`git diff --cached` 已经暂存起来的文件和上次提交时的快照之间的差异；  
`git diff HEAD` 工作目录中的文件和上次提交之间的改动；  
`git diff [version tag]` 查看指定版本之后的改动；  

## branch,tag相关

查看本地分支 `git branch`  
查看远程分支 `git branch -r`  
查看所有分支 `git branch -a`  

本地创建新分支 `git branch <branchName>`  
切换到分支xx `git checkout <branchName>`  
创建分支的同时切换到该分支上 `git checkout -b <branchName>`  
合并某分支到当前分支`git merge <branchName>`

将新分支推送到远程repo上 `git push origin <branchName>`  

删除本地分支 `git branch -d <branchName>`  
删除远程分支 `git push origin --delete <branchName>`  

删除远程tag `git push origin --delete tag <tagname>`  
把本地tag推送到远程`git push --tags`  
获取远程tag`git fetch origin tag <tagname>`  

**重命名远程分支:**  
即先删除远程分支，然后重命名本地分支，再重新提交一个远程分支

```sh
git push --delete origin <branchName> #删除远程分支
git branch -m b1 new_b  #重命名本地分支
git push origin new_b   #推送本地分支
```

**切换分支**如架构调整之类需要将先前代码以一个分支留存继续在此分支开发

```sh
git branch -m thisBranch oldBranch
git push origin oldBranch  #留存老版到仓库
git checkout thisBranch  #切换thisBranch继续开发
```

**删除不存在对应远程分支的本地分支:**  
使用`git remote show origin`查看分支的状态，看到\<branch>是stale的，使用`git remote prune origin`可以将其从本地版本库中去除
>更简单的方法是使用`git fetch -p`，它在fetch之后删除掉没有与远程分支对应的本地分支

有时候我们需要**创建一个干净的分支**，其不继承任何提交没有父节点，而上文的 `git checkout xx` 创建的分支xx是有父节点的，包含了历史提交。流程如下：
1, 创建干净的分支 `git checkout --orphan xx` ；  
2, 删除工作目录中其他分支存在的内容 `git rm -rf .` ;  
3, 给分支xx添加内容 `git add file1 file2...` ;  
4, 提交到本地仓库 `git commit -m "something"` ;  
5, 推送到远程仓库 `git push origin xx` ;  

**其他分支的某个commit并入本分支**当我们需要将其他分支的某一次提交合入到本地当前分支上，那么就要使用`git cherry-pick commitid`
> 如果在git cherry-pick后加一个分支名，则表示将该分支顶端提交进cherry-pick如：`git cherry-pick <branchname>`

* git cherry-pick ..\<branchname\>和git cherry-pick ^HEAD \<branchname\>

以上两个命令作用相同，表示应用所有提交引入的更改，这些提交是branchname的祖先但不是HEAD的祖先(即当前分支)，比如，现在我的仓库中有三个分支，其提交历史如下图：

```log
               C<---D<---E  branch2
              /
master   A<---B  
              \
               F<---G<---H  branch3
                         |
                         HEAD
```

如果我使用`git cherry-pick ..branch2`或者`git cherry-pick ^HEAD branch2`,那么会将属于branch2的祖先但不属于branch3的祖先的所有提交引入到当前分支branch3上，并生成新的提交，执行后的提交历史如下：

``` log

               C<---D<---E  branch2
              /
master   A<---B  
              \
               F<---G<---H<---C<---D<---E  branch3
                                        |
                                        HEAD
```

## 撤销相关

git撤销操作主要有如下方式:
撤销上一次提交，并将暂存区文件重新提交 `git commit --amend` ;  
拉取暂存区文件替换工作区文件 `git checkout file1 dir1...` ;  
拉取版本库文件到暂存区, 不影响工作区 `git reset HEAD file1 dir1...` ;  

> 第一种情况下--amend会打开编辑文件其中快捷键^表示ctrl, M表示alt(linux)  

meta键是以前MIT计算机键盘上的的一个特殊键

## 更换地址

查看远程仓库信息 `git remote -v`  
本地仓库更换远程仓库地址

``` bash
git remote rm origin
git remote add origin new_addr
#等价操作
git remote set-url origin new_addr
```

## 删除和重命名

``` bash
git rm xxx          #将文件从索引和工作目录中都删除
git rm --cached xxx #删除索引中的文件并把它保留在工作目录中
git checkout xxx    #文件删除后的恢复
git mv f1  f2       #重命名,f2不存在
```

## 多仓库工作

github和码云同时维护, 代码在github上

``` bash
cd repositoriesDir
git remote -v
#origin https://github.com/yoloz/abc.git (fetch)
#origin https://github.com/yoloz/abc.git (push)
git remote add def https://gitee.com/user/def.git
git remote -v
#def https://gitee.com/user/def.git (fetch)
#def https://gitee.com/user/def.git (push)
#origin https://github.com/yoloz/abc.git (fetch)
#origin https://github.com/yoloz/abc.git (push)
git pull def master:def  #pull def中的master分支到本地def分支
git checkout def #change branch to def
git merge master [--allow-unrelated-histories] #拷贝本地master分支到本地def分支中
git push def def:master  #push 本地def分支到def中的master分支
```

> fatal: refusing to merge unrelated histories添加

`--allow-unrelated-histories` 告诉git允许不相关历史合并  
`git pull def master:def` 用于新建分支，如果更新def分支，则要先checkout到def分支

## 修改commit

合并多个commit为一个完整commit, 或者多分支合并时去除被合并分支的一些commit

``` sh
git log
#commit f711d30598620669a692a8115d1c798af66da311
#Author: abc <abc@gmail.com>
#Date:   Thu Jul 11 11:38:15 2019 +0800
#    commit提交说明
git rebase -i f711d30 #commit标志的前7位

#pick e157f87 Initial commit
#pick f711d30 2019-07-11 11:38:15
#pick e04d2b0 2019-07-11 11:40:52

# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit

```

> * `git rebase -i  [startpoint]  [endpoint]` 其中-i的意思是--interactive，即弹出交互式的界面让用户编辑完成合并操作，[startpoint]  [endpoint]则指定了一个编辑区间，如果不指定[endpoint]，则该区间的终点默认是当前分支HEAD所指向的commit(注：该区间指定的是一个前开后闭的区间)。
> * ^X的^表示ctrl, M-A的M表示alt
> * 修改后(如将pick换成d), ctrl+x退出, 提示是否保存修改，选择yes, 然后选择alt+b(backup file), 然后enter回车即可
> * 修改conflict，然后push

## 克隆部分文件

Sparse Checkout模式:  
1.指定远程仓库

``` sh
mkdir project_folder
cd project_folder
git init
git remote add -f origin <url>
```

2,指定克隆模式`git config core.sparsecheckout true`  
3,指定克隆的文件夹(或者文件)

```sh
echo “libs” >> .git/info/sparse-checkout
echo “apps/register.go” >> .git/info/sparse-checkout
echo “resource/css” >> .git/info/sparse-checkout
```

4,拉取远程文件`git pull origin master`

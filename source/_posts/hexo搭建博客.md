---
title: hexo搭建博客
comments: false
toc: false
date: 2018-11-19 11:26:03
categories: hexo
tags:
---
## 环境准备

### 搭建nodejs环境
---
初次需要创建仓库【username】.github.io;  
创建一个分支存放原始文件，方便跨机操作；

---
### 拉取远程仓库到本地并切换到存放原始文件的【分支】
```bash
git clone http://github.com/user/user.github.io 【仓库目录】
cd 【仓库目录】
git branch 【分支】
git checkout 【分支】
```
### 安装hexo  

 `npm install hexo-cli -g`  
 
---
初次搭建需初始化:  
``` bash 
hexo init 【临时目录】
cp -rf 【临时目录】/* 【仓库目录】
```
---
### 继续完成安装
```bash
cd 【仓库目录】
npm install
npm install hexo-deployer-git
```
执行`hexo server`通过浏览器即可访问博客了;

## 自定义
打开themes/landscape中修改相关文件，具体可参考互联网;

## hexo的使用
可参考上篇[《start-hexo》](/2018/11/19/start-hexo/)

## 问题
`hexo generate --deploy`报错:  
```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```
如果不想设置全局变量，可以单独设置:
```bash
cd .deploy_git/
git config user.name "Your Name"
git config user.email "you@example.com"
```
---
title: hexo搭建博客
comments: false
toc: false
date: 2018-11-27 14:54:43
categories: hexo
tags:
---

## 环境准备

### 搭建nodejs环境

---
初次需要创建仓库【username】.github.io;
创建一个分支存放原始文件，方便跨机操作；

---
<!-- more -->

### 拉取远程仓库【分支】到本地

```bash
git clone -b 【分支】 https://github.com/user/user.github.io.git 【仓库目录】
```

### 安装hexo  

```bash
npm install hexo-cli -g
```

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
```

初次git发布还需安装: `npm install hexo-deployer-git`

执行 `hexo server` 通过浏览器即可访问博客了;

## 自定义

打开themes/landscape中修改相关文件，具体可参考互联网;

## hexo的使用

git部署发布: `hexo generate --deploy`

更多信息参考上篇[《start-hexo》](/2018/11/19/start-hexo/)

## 问题

 `hexo generate --deploy` 报错:  

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

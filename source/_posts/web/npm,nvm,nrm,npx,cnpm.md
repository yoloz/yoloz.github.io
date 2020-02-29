---
title: npm,nvm,nrm,npx,cnpm
comments: false
toc: false
date: 2020-02-29 10:31:18
categories: web
tags:
---

Node.js 是一个基于 Chrome V8 引擎的 JavaScript 运行环境

## npm

npm 的全称是 Node Package Manager 是 JavaScript 世界的包管理工具,并且是 Node.js 平台的默认包管理工具。通过 npm 可以安装、共享、分发代码,管理项目依赖关系。

### npm install本地安装

将安装包放在./node_modules 下（运行 npm 命令时所在的目录），如果没有node_modules目录，会在当前执行npm命令的目录下生成node_modules目录。

可以通过 require() 来引入本地安装的包。

### npm install -g全局安装

将安装包放在/usr/local下或者你 node 的安装目录。

可以直接在命令行里使用。

### npm install --save

* 会把包安装到node_modules目录中
* 会在package.json的dependencies属性下添加包
* 之后运行npm install命令时，会自动安装包到node_modules目录中
* 之后运行npm install --production或者注明NODE_ENV变量值为production时，会自动安装包到node_modules目录中

### npm install --save-dev

* 会把包安装到node_modules目录中
* 会在package.json的devDependencies属性下添加包
* 之后运行npm install命令时，会自动安装包到node_modules目录中
* 之后运行npm install --production或者注明NODE_ENV变量值为production时，不会自动安装包到node_modules目录中

## nvm

在开发中，有时候对node的版本有要求，有时候需要切换到指定的node版本来重现问题等。遇到这种需求的时候，我们需要能够灵活的切换node版本。nvm就是为解决这个问题而产生的，他可以方便的在同一台设备上进行多个node版本之间切换

``` sh
npm install -g nvm
nvm install v6.2.0(或者6.2)  #安装指定版本，可模糊安装
nvm uninstall xxx           #删除已安装的指定版本，语法与install类似
nvm use                     #切换使用指定的版本node
```

> nvm不支持Windows，有替代品nvm-windows

## n

n是node一个模块，可以用来管理和切换node版本，其作者是TJ Holowaychuk（出名的Express框架作者），使用非常之简单

``` sh
npm install -g n
n                     #查看已安装版本
n latest              #安装最新版本并使用
n latest -d           #下载最新版但不使用，-d参数表示为仅下载
n stable              #安装最新稳定版本并使用
n <version>           #安装某个版本并使用
n rm <version ...>    #删除某些版本
```

## nrm

nrm(npm registry manager)是npm资源管理器，允许你快速切换npm源

由于网络原因，npm在国内使用比较慢，所以建议切换npm源到国内镜像或者使用cnpm

* 方式一

``` sh
npm install -g nrm
nrm ls
#* npm -----  https://registry.npmjs.org/
#  cnpm ----  http://r.cnpmjs.org/
#  taobao --  https://registry.npm.taobao.org/
#  nj ------  https://registry.nodejitsu.com/
#  rednpm -- http://registry.mirror.cqupt.edu.cn
#  skimdb -- https://skimdb.npmjs.com/registry
nrm use cnpm
#nrm del cnpm
```

* 方式二
``` sh
npm install -g cnpm --registry=http://r.cnpmjs.org/
```

## cnpm

因为npm安装插件是从国外服务器下载，受网络影响大，可能出现异常。来自官网：“这是一个完整 npmjs.org 镜像，你可以用此代替官方版本(只读，不能发布自己的包或注册用户)，同步频率目前为10分钟一次以保证尽量与官方服务同步。”

``` sh
npm install cnpm -g --registry=http://r.cnpmjs.org/
```

> 安装完后最好查看其版本号cnpm -v或关闭命令提示符重新打开，安装完直接使用有可能会出现错误
>
> cnpm跟npm用法完全一致，只是在执行命令时将npm改为cnpm。

## npx

npm v5.2.0 引入的一条命令（npx），npx 会帮你执行依赖包里的二进制文件。引入这个命令的目的是为了提升开发者使用包内提供的命令行工具的体验。全局安装 parcel，但有时不同项目使用不同版本，不允许使用全局包，只能考虑下面一些方法。使用 npm scripts，在 package.json 加一个 script ,将 node_modules 的可执行目录加到 PATH 中.指定可执行命令路径。当我们执行 npx parcel index.html 时，会自动去./node_modules/.bin 目录下搜索


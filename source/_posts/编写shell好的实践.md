---
title: 编写shell好的实践
comments: false
toc: false
date: 2018-12-04 14:03:49
categories: shell
tags:
---
## 开头有“shebang”
脚本第一行以"#!"开头的注释，标明在没有指定解释器的时候默认的解释器，一般如`#!/bin/bash`,解释器有很多，除bash之外可以用命令查看本机支持的解释器
``` bash
$ cat /etc/shells
# /etc/shells: valid login shells
/bin/sh
/bin/dash
/bin/bash
/bin/rbash
```
<!--more-->
当我们直接使用./xxx.sh执行脚本的时候，如果没有shebang,则默认用$SHELL指定的解释器，否则用shebang指定的解释器。然而上面的写法可能有些环境不兼容，可以使用如下方式来指定shebang
``` bash
#!/usr/bin/env bash
```

## 太长要分行
在调用某些程序的时候，参数可能会很长，这时候为了较好的阅读体验，可以用反斜杠来分行
``` bash
./configure \
-prefix=/usr \
-sbin-path=/usr/sbin/nginx \
-conf-path=/etc/nginx/nginx.conf \
```
> 注意在反斜杠前有个空格

## 多用双引号
推荐在使用"$"来获取变量的时候加上双引号，否则在一些情况下会出现麻烦，如下
``` bash
#!/usr/bin/env bash
#当前文件下有a.sh文件
var="*.sh"
echo $var
echo "$var"
```
运行结果
```
a.sh
*.sh
```

## 脚本结构化
具体功能使用函数定义，然后入口调用，如下：
``` bash
#!/usr/bin/env bash
func1(){
    #do sth
}
func2(){
    #do sth
}
main(){
    func1
    func2
}
main "$@"
```

## 作用域
shell中默认的变量作用域是全局的，如下：
``` bash
#!/usr/bin/env bash
var=1
func(){
    var=2
}
func
echo "$var"
```
输出结果是2不是1。因此相比直接使用全局变量，可以使用`local readonly declare`来定义声明变量。

## 函数返回值
shell中函数的返回值只能是整数，可以通过echo或者print之类的传递字符串，如下
``` bash
func1(){
    echo "2333"
}
res=$(func1)
echo "This is from $res."
```

## 间接引用
``` bash
#!/usr/bin/env bash
var1="1234"
var2="var1"
```
现在需要通过var2来获取var1的值，可以：  
* 通过eval命令`eval echo \$$var2`，不推荐使用
* 变量名前加一个!`echo ${!var2}`  

上面的这种只能做到取值，如果要赋值还是用eval来处理
``` bash
var1=var2
eval $var1="1234"
echo "$var2"
```

## 巧用heredocs
一种多行输入方法，在"<<"后定一个标识符，接着我们可以输入多行内容，直到再次遇到标识符为止。定义形式为
```
命令 << HERE
...
...
HERE
```
使用heredocs，可以较方便的生成一些模板文件，如下:
``` bash
cat >> /etc/rsyncd.conf << EOF
log file = /usr/local/logs/rsyncd.log
log format = %t %m %f %b
EOF
```
> * HERE只是一个标识，可以换成任意其他合法字符；
> * 结束的标识符一定要顶格写，前面不能有任何字符,后面也不能有任何字符，包括空格;
> * 起始的标识符前后空格会被省略；

其他用法：  
* 定义变量
``` bash
var1="just test"
var2=`cat << H
line 1
line 2
$var1
H`
echo "$var2"
```
结果
```
line 1
line 2
just test
```
* 定义一段批注，单行批注(#注释)
``` bash
:<<HERE
批注一
批注二
...
HERE
```
> :代表什么都不做

## 获取路径
很多情况下需要获取脚本的路径，然后以此路径为基准去找其他的路径。直接使用`pwd`获取是不严谨的，其获得是当前shell的执行路径，并不是当前脚本的执行路径。正确做法如下:
``` bash
script_dir=$(cd `dirname $0` && pwd)
script_dir=$(dirname `readlink -f $0`)
```
> readlink print value of a symbolic link or canonical file name(输出符号链接值或者权威文件名)
> ``` bash
> $ readlink /usr/bin/awk  
> /etc/alternatives/awk  #这个还是一个符号连接 
> $ readlink /etc/alternatives/awk  
> /usr/bin/gawk  #这个是真正的可执行文件  
> ########################################
> $ readlink -f /usr/bin/awk  
> /usr/bin/gawk 
> ```
> -f 递归符号链接,直到非符号链接的文件位置,限制是最后必须存在一个非符号链接的文件

## 简短与效率
原则上能一条命令解决的问题不用多条命令解决。  
`cat /etc/passwd | grep root`==>`grep root /etc/passwd`  
简短有时候还能提升效率，如下
``` bash
find . -name "*.txt" |xargs sed -i s/233/666/g
find . -name "*.txt" |xargs sed -i s/235/626/g
find . -name "*.txt" |xargs sed -i s/333/616/g
find . -name "*.txt" |xargs sed -i s/233/664/g
```
``` bash
find . -name "*.txt" |xargs sed -i "s/233/666/g;s/235/626/g;s/333/616/g;s/233/664/g"
```
这两种方法做的事情一样，查找所有的.txt后缀的文件并做一系列替换。前者是多次执行find，后者是执行一次find，但是增加了sed的模式串。第一种更直观一点，但是当替换的量变大的时候，第二种的速度就会比第一种快很多。这里效率提升的原因，就是第二种只要执行一次命令，而第一种要执行多次。并且，巧用xargs命令，我们还可以十分方便的进行并行化处理:
``` bash
find . -name '*.txt' |xargs -P $(nproc) sed -i "s/233/666/g;s/235/626/g;s/333/616/g;s/233/664/g"
```
> -P 指定并行度，进一步加快执行效率。

## 并行化
shell中最简单的并行化是通过”&”以及”wait”命令。
``` bash
func1(){
    #do sth
｝
for((i=0;i<10;i++))do
    func1 &
done
wait
```
当然，这里并行的次数不能太多，否则机器会卡死。如果图省事可以使用parallel命令来做，或者是用上面提到的xargs来处理。

## 一些建议
* 使用func(){}来定义函数，而不是func{}
* 使用[[]]来代替[]
* 使用$()将命令的结果赋给变量，而不是反引号
* 使用printf代替echo进行回显
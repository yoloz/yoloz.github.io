---
title: java,javac,jar命令
comments: false
toc: false
date: 2018-11-28 14:54:43
categories: java
tags:
---

## 简单的jar

生成的jar包结构:  

``` txt
META-INF  
Tom.class  
Hello.class(主类)  
xxxxx.class
```

方法步骤:  
1, 定义Main-Class  
创建文件夹META-INF, 创建文件MENIFEST. MF, 內容如下:

``` txt
Manifest-Version: 1.0
Main-Class: Hello

```

> 注意冒号后面有一个空格, 整个文件最后有一行空行  

2, 编译及打包  

``` bash
javac Hello.java  #编译
jar -cvfm hello.jar META-INF\MENIFEST.MF Hello.class Tom.class xxx.class
```

> c表示要创建一个新的jar包;

v表示创建的过程中在控制台输出创建过程的一些信息;
f表示给生成的jar包命名;
m表示自定义MENIFEST文件  

## 有目录结构的jar

生成的jar包结构:  

``` txt
META-INF  
com  
   Tom.class  
   xxx.class
Hello.class(主类)
```

方法步骤:  
1, 定义Main-Class  
略(同上)

2, 编译及打包  

``` bash
javac Hello.java -d target  #编译
cd target
mv -r xxx/META-INF .
jar -cvfm hello.jar META-INF\MENIFEST.MF *
```

> 最后的*表示把当前目录下所有文件都打在jar包里  

## 依赖第三方jar

方法步骤:  
1, 定义Main-Class  
略(同上)

2, 编译及打包  

``` bash
javac -cp xxx.jar Hello.java -d target  #编译
cd target
mv -r xxx/META-INF .
jar -cvfm hello.jar META-INF\MENIFEST.MF *
```

> -cp 表示 -classpath  

对于 `java -jar hello.jar` 报错ClassNotFoundException:xxxxx，可以 `java -cp xxx.jar -jar hello.jar` 或者在Main-Class中添加classPath:  

``` txt
Manifest-Version: 1.0
Main-Class: Hello
Class-Path: xxx.jar #路径指向你需要的所有jar包

```

> 上述中xxx.jar要和hello.jar在同一路径下，执行 `java -jar hello.jar` 即可

## javac和java

``` bash
javac -classpath $(echo ${dir}/lib/*.jar | tr ' ' ':') com/xxx/Test.java
java -cp .:$(echo ${dir}/lib/*.jar | tr ' ' ':') com.xxx.Test
```

> tr是个简单的替换命令，从标准输入中替换、缩减和/或删除字符，并将结果写到标准输出

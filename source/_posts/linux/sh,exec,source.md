---
title: sh, exec, source
comments: false
toc: false
date: 2019-09-18 23:50:33
categories:
tags:
---

### sh方式

使用 `$ sh script.sh` 执行脚本时，当前shell是父进程，生成一个子shell进程，在子shell中执行脚本。脚本执行完毕，退出子shell，回到当前shell.

> `$ ./script.sh` 与 `$ sh script.sh` 等效
> 通过sh执行脚本时，修改的上下文不会影响当前shell

### source方式

使用 `$ source script.sh` 方式，在当前上下文中执行脚本，不会生成新的进程。脚本执行完毕，回到当前shell.

> `source` 方式也叫点命令， `$ . script.sh` 与 `$ source script.sh` 等效。注意在点命令中， `.` 与 `script.sh` 之间有一个空格。
> 通过source执行脚本时，修改的上下文会影响当前shell

### exec方式

使用 `exec command` 方式，会用 `command` 进程替换当前shell进程，并且保持 `PID` 不变。执行完毕，直接退出，不回到之前的shell环境。


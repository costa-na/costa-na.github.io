---
layout: post
title: MIT 6.828课程解析（三） Lecture 1：sh.c分析
---

## 简介
补全[sh.c](https://pdos.csail.mit.edu/6.828/2014/homework/sh.c)缺失的功能是6.828第一个[homework](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)，其目标是通过实现shell的部分功能让学生熟悉Unix的系统调用接口和shell的使用。

sh.c这个文件实现一个简化版的shell，其默认提供了大部分框架代码，需要手动补充的缺失的功能是：
* 实现单条命令的执行
* 实现重定向（redirect）功能
* 实现管道（pipe）功能

在动手写代码之前，让我们先浏览一遍整个sh.c的框架，力图理解其完整的调用流程。


## 分析
sh.c只是一个普通的c代码文件，其入口函数和其他c代码一样，都是`main`函数，所以让我们先从`main`函数开始，作为整个分析的入口点

### `main`
sh.c的`main`函数极其简单，在创建了一个命令读取的缓存之后，就进入一个“读命令” > “执行命令” > “读下一条命令” > “执行命令” > ...的循环中，直到读取命令返回失败（读到`EOF`），然后就使用`exit(0)`结束自身进程。其流程图如下：

![main](/public/img/main.png)

要点：

* 命令缓存是由`main`在静态分配内存中开辟的一个100字节的char型数组（[What does “static” mean in a C program?](http://stackoverflow.com/questions/572547/what-does-static-mean-in-a-c-program)）
* 如果读取命令成功，`main`会在调用`fork`之前判断用户输入的命令是否为cd，为何要在父进程（`main`自身）而不是在执行命令的子进程中来判断，是因为cd命令会改变当前进程的工作目录（[Working directory](https://en.wikipedia.org/wiki/Working_directory)），从而改变后续命令的执行环境，所以需要在父进程中执行cd
* `main`对cd的处理写得不是很好，这里存在一个隐藏的bug，如果在用户输入的命令之后存在空格，如|cd /bin  |，`chdir`将执行失败，因此这里需要将命令结尾处的空白字符strip掉
* `main`在调用`fork`之后会使用`wait`来等待子进程执行结束，所以改变此处逻辑，让`main`放下子进程后，不等待立即返回，可以使sh.c支持后台执行（这个是这个homework的challenge之一）
* `main`并没有直接调用`fgets`来获取用户输入，而是将`fget`封装在了`getcmd`中，这样可以在读取用户输入之前，更改输出的提示符

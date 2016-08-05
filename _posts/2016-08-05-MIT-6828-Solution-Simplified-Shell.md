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
sh.c的`main`函数极其简单，在创建了一个100字节的字符数组作为命令读取的缓存之后，就进入一个“读命令” -> “执行命令” -> “读下一条命令” -> “执行命令” -> ...的循环中，直到读取命令返回失败（读到`EOF`），然后就使用`exit(0)`结束自身进程。其流程图如下：

![main](/_posts/img/main.png)

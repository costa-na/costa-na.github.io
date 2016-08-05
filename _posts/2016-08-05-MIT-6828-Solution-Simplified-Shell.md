---
layout: post
title: MIT 6.828课程解析（三） Lecture 1：sh.c分析
---

## 简介
补全[sh.c](https://pdos.csail.mit.edu/6.828/2014/homework/sh.c)缺失的功能是6.828第一个[homework](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)，其目标是通过实现shell的部分功能让学生熟悉Unix的系统调用接口和shell的使用。

sh.c这个文件实现一个简化版的shell，其默认提供了大部分框架代码，需要手动补充的缺失的功能是：
* `runcmd`函数中，对于执行单条命令的处理

  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);
    fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    break;

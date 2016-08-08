---
layout: post
title: MIT 6.828课程解析（五） Lecture 1：sh.c功能补全
---

## 简介
补全[sh.c](https://pdos.csail.mit.edu/6.828/2014/homework/sh.c)缺失的功能是6.828第一个[homework](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)，其目标是通过实现shell的部分功能让学生熟悉Unix的系统调用接口和shell的使用。

我们在上一篇（[MIT 6.828课程解析（四） Lecture 1：sh.c分析](https://costa-na.github.io/2016/08/08/MIT-6828-Solution-Simplified-Shell/)）已经详细分析了sh.c的结构和逻辑了，在分析的最后，我们可以看到`runcmd`这个函数缺少对三种不同命令的处理逻辑，因此，我们在这篇文章中将会补全这些缺失的逻辑代码，让sh.c可以完整的编译运行起来。

## 功能实现

### 运行独立命令（`execcmd`）
上篇我们已经介绍了可执行命令（`execcmd`）、重定向命令（`redircmd`）和管道命令（`pipecmd`）的区别。其中，可执行命令是重定向命令和管道命令的基础，其实现也非常简单，所以让我们先来实现可执行命令的执行逻辑。

在`runcmd`中，对于`execcmd`类型的命令的处理分支是这样的：

  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);
    fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    break;

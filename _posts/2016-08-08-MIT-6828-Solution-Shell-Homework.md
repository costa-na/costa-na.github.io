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

```
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);
    fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    break;
```

如果之前已经阅读过Ritchie和Thompson写的[The UNIX Time-Sharing System](https://pdos.csail.mit.edu/6.828/2014/readings/ritchie78unix.pdf)，那么这里实现就非常直接了，即使之前没有阅读的这篇文档，这里也推荐翻阅一下，毕竟是Unix作者亲自写的介绍。
在Ritchie和Thompson这篇介绍中已经提到了`fork`和`execute`这两个系统调用（system call），在xv6手册（[xv6: a simple, Unix-like teaching operating system](https://pdos.csail.mit.edu/6.828/2014/xv6/book-rev8.pdf)）中的chapter 0中也直接提到了如何在`runcmd`中通过调用`exec`来执行命令：

> `runcmd` (8406) runs the actual command. For "echo hello", it would call `exec` (8426). If exec succeeds then the child will execute instructions from echo instead of runcmd. 

因此，在这个`case`分支，我们只需要调用`exec`类系统调用，将欲执行的命令字符串和其参数数组传递给它，就可以完成命令的执行了。这里需要注意的是，由于sh.c是编译在Linux环境下面的，因此这里需要从[`exec`](http://pubs.opengroup.org/onlinepubs/009695399/functions/exec.html)家族函数中选一个合适的函数来执行此功能，`exec`类系统调用一共有六种：`execl`、`execv`、`execle`、`execve`、`execlp`和`execvp`，因为我们要传递给的是可执行文件的具体路径（`path`）和一个`char`指针数组（`argv`），所以这里选择了`execv`，补全后的代码如下：

```
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);
    if(execv(ecmd->argv[0], ecmd->argv) < 0){
      perror("execv");
      exit(0);
    }
    break;
```

在上篇我们已经说过，在`execcmd`的`argv`数组中存放的是命令的调用参数，其第一个元素存放的是命令自身的可执行文件的路径，所以这里我们将`ecmd->argv[0]`作为第一个参数传递给了`execv`。我们需要判断`execv`是否执行成功，如果执行失败了，就是使用[perror](http://pubs.opengroup.org/onlinepubs/009695399/functions/perror.html)打印错误信息，然后使用`exit(0)`结束进程。

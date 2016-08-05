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

1. 命令缓存是由`main`在静态分配内存中开辟的一个100字节的char型数组（[What does “static” mean in a C program?](http://stackoverflow.com/questions/572547/what-does-static-mean-in-a-c-program)）
2. 如果读取命令成功，`main`会在调用`fork`之前判断用户输入的命令是否为cd，为何要在父进程（`main`自身）而不是在执行命令的子进程中来判断，是因为cd命令会改变当前进程的工作目录（[Working directory](https://en.wikipedia.org/wiki/Working_directory)），从而改变后续命令的执行环境，所以需要在父进程中执行cd
3. `main`对cd的处理得不是很好，这里存在一个隐藏的bug，如果在用户输入的命令之后存在空格，如“cd /bin  ”，`chdir`将执行失败，因此这里需要将命令结尾处的空白字符strip掉
4. `main`在调用`fork`之后会使用`wait`来等待子进程执行结束，所以改变此处逻辑，让`main`放下子进程后，不等待立即返回，可以使sh.c支持后台执行（这个是这个homework的challenge之一）
5. `main`并没有直接调用`fgets`来获取用户输入，而是将`fget`封装在了`getcmd`中，这样可以在读取用户输入之前，更改输出的提示符

### `getcmd`
`getcmd`只是简单封装了`fgets`，其主要功能是：

1. 判断`stdin`是否为终端设备，是就输出提示符“6.828$ ”，如果要将shell的提示符更改为其他形式就可以修改此处
2. 使用`memset`清空命令缓存数组`buf`
3. 调用`fgets`从`stdin`读取一行用户输入（以回车符`\n`结束），将其存到`buf`中，这里需要参考`fgets`这个函数的说明文档（[fgets](http://pubs.opengroup.org/onlinepubs/009695399/functions/fgets.html)）
  * **注意**：`fgets`读取的最大字符数比第二个参数指定的要少一个，这是因为要在最后一个位置上填上`NULL`表示字符串的结束
4. 判断读取是否成功，成功返回0，未成功即读到`EOF`返回-1，这个是Unix的惯用法（成功返回0，否则返回-1）

### `fork1`
`fork1`封装了`fork`，这样在`fork`执行失败（返回-1）的时候，可以调用`perror`打印错误信息，便于使用和调试

至此`main`的主体逻辑已经分析完毕，接下来分析命令的解析和具体执行逻辑，这部分占了sh.c的大部分代码。

### `parsecmd`预览
`parsecmd`从名字上可以推断，该函数主要执行对对用户输入的命令字符的解析，`parsecmd`主体逻辑并不复杂，但此时我们只需要概括的看一下它的执行流程，并不会深入到每个子函数中去。

1. 通过`strlen`定位到命令字符串结尾后面的第一个位置上
2. 将字符串的起始位置和结尾后面的第一个位置传给`parseline`，返回值赋给`struct cmd`类型的变量`cmd`
3. 再将相同的参数连通一个空字串传给`peek`函数
4. 判断命令行是否已经解析完毕，如果还有剩余的字符，打印错误信息，结束进程
5. 将`cmd`返回给`runcmd`

到这里我们大致了解了`parsecmd`在做什么：解析命令字符串，将其转化成一个`cmd`结构，然后交给`runcmd`执行。现在让我们了解一下几个工具函数的实现，这样在后面再次遇到的时候就不用再去查看了。

### `peek`

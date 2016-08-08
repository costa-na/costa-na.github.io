---
layout: post
title: MIT 6.828课程解析（五） Lecture 1：sh.c功能补全
---

## 简介
补全[sh.c](https://pdos.csail.mit.edu/6.828/2014/homework/sh.c)缺失的功能是6.828第一个[homework](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)，其目标是通过实现shell的部分功能让学生熟悉Unix的系统调用接口和shell的使用。

我们在上一篇（[MIT 6.828课程解析（四） Lecture 1：sh.c分析](https://costa-na.github.io/2016/08/08/MIT-6828-Solution-Simplified-Shell/)）已经详细分析了sh.c的结构和逻辑了，在分析的最后，我们可以看到`runcmd`这个函数缺少对三种不同命令的处理逻辑，因此，我们在这篇文章中将会补全这些缺失的逻辑代码，让sh.c可以完整的编译运行起来。

## 功能实现

### 运行可执行命令（`execcmd`）
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

### 运行重定向命令（`redircmd`）
在`runcmd`中，对于`execcmd`类型的命令的处理分支是这样的：

```
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    fprintf(stderr, "redir not implemented\n");
    // Your code here ...
    runcmd(rcmd->cmd);
    break;
```

在xv6手册中已经提到了如何实现重定向命令的执行：

> This behavior allows the shell to implement I/O redirection by forking, reopening chosen file descriptors, and then execing the new program. Here is a simplified version of the code a shell runs for the command `cat <input.txt`:
```
  char *argv[2];
  argv[0] = "cat";
  argv[1] = 0;
  if(fork() == 0) {
    close(0);
    open("input.txt", O_RDONLY);
    exec("cat", argv);
  }
```
After the child closes file descriptor 0, `open` is guaranteed to use that file descriptor for the newly opened `input.txt`: 0 will be the smallest available file descriptor. `Cat` then executes with file descriptor 0 (standard input) referring to `input.txt`.

这里需要知道[`close`](http://pubs.opengroup.org/onlinepubs/009695399/functions/close.html)关闭并回收一个文件描述符（`fd`），让其能够被之后的`open`调用使用：

> The `close()` function shall deallocate the file descriptor indicated by `fildes`. To deallocate means to make the file descriptor available for return by subsequent calls to `open()` or other functions that allocate file descriptors. 

然后还要知道[`open`](http://pubs.opengroup.org/onlinepubs/009695399/functions/open.html)是分配一个文件描述符（`fd`），并将其和指定的文件相关联。`open`分配的文件描述符是文件描述符表中最小的可用文件描述符：

> The `open()` function shall return a file descriptor for the named file that is the lowest file descriptor not currently open for that process. 

因为要重定向的文件描述符已经存在于`rcmd->fd`中，所以这里我们先使用`close(rcmd->fd)`关闭欲重定向的文件描述符，再使用`open(rcmd->file, ...)`打开。`open`的第二个参数指定打开文件的`mode`，这个值也已经存放在`rcmd->mode`中了，直接传递给`open`即可。当指定打开的文件不存在时，`open`会新建一个文件然后在尝试打开，其第三个参数即指定了新建文件的属性，这里我们使用了[`<sys/stat.h>`](http://pubs.opengroup.org/onlinepubs/7908799/xsh/sysstat.h.html)中的两个常量来指定：

* `S_IRWXU`：ower具有读写和可执行权（read, write, execute/search by owner）
* `S_IRGRP`:group具有可读权（read permission, group）
* 
因此，补全后的代码如下：

```
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    fprintf(stderr, "redir not implemented\n");
    close(rcmd->fd);
    if(open(rcmd->file, rcmd->mode, S_IRWXU|S_IRGRP) < 0){
      perror("open");
      exit(0);
    }
    runcmd(rcmd->cmd);
```

    break;

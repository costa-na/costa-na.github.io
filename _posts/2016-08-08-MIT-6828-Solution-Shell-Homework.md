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

因此，在这个`case`分支，我们只需要调用`exec`类系统调用，将欲执行的命令字符串和其参数数组传递给它，就可以完成命令的执行了。这里需要注意的是，由于sh.c是编译在Linux环境下面的，因此这里需要从[`exec`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/exec.html)家族函数中选一个合适的函数来执行此功能，`exec`类系统调用一共有六种：`execl`、`execv`、`execle`、`execve`、`execlp`和`execvp`，因为我们要传递给的是可执行文件的具体路径（`path`）和一个`char`指针数组（`argv`），所以这里选择了`execv`，补全后的代码如下：

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

在上篇我们已经说过，在`execcmd`的`argv`数组中存放的是命令的调用参数，其第一个元素存放的是命令自身的可执行文件的路径，所以这里我们将`ecmd->argv[0]`作为第一个参数传递给了`execv`。我们需要判断`execv`是否执行成功，如果执行失败了，就是使用[perror](http://pubs.opengroup.org/onlinepubs/9699919799/functions/perror.html)打印错误信息，然后使用`exit(0)`结束进程。

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

这里需要知道[`close`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/close.html)关闭并回收一个文件描述符（`fd`），让其能够被之后的`open`调用使用：

> The `close()` function shall deallocate the file descriptor indicated by `fildes`. To deallocate means to make the file descriptor available for return by subsequent calls to `open()` or other functions that allocate file descriptors. 

然后还要知道[`open`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html)是分配一个文件描述符（`fd`），并将其和指定的文件相关联。`open`分配的文件描述符是文件描述符表中最小的可用文件描述符：

> The `open()` function shall return a file descriptor for the named file that is the lowest file descriptor not currently open for that process. 

因为要重定向的文件描述符已经存在于`rcmd->fd`中，所以这里我们先使用`close(rcmd->fd)`关闭欲重定向的文件描述符，再使用`open(rcmd->file, ...)`打开。`open`的第二个参数指定打开文件的`mode`，这个值也已经存放在`rcmd->mode`中了，直接传递给`open`即可。当指定打开的文件不存在时，`open`会新建一个文件然后在尝试打开，其第三个参数即指定了新建文件的属性，这里我们使用了[`<sys/stat.h>`](http://pubs.opengroup.org/onlinepubs/9699919799/xsh/sysstat.h.html)中的两个常量来指定：

* `S_IRWXU`：ower具有读写和可执行权（read, write, execute/search by owner）
* `S_IRGRP`:group具有可读权（read permission, group）
* 
因此，补全后的代码如下：

```
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    close(rcmd->fd);
    if(open(rcmd->file, rcmd->mode, S_IRWXU|S_IRGRP) < 0){
      perror("open");
      exit(0);
    }
    runcmd(rcmd->cmd);
```

    break;

### 运行管道命令（`pipecmd`）
管道命令因为涉及到较多的文件描述符操作，所以较为复杂，我们一步一步来分析。首先还是先参阅xv6手册，其中也提到了关于pipe命令的实现方法：

> The following example code runs the program wc with standard input connected to the read end of a pipe.
```
  int p[2];
  char *argv[2];
```
```
  argv[0] = "wc";
  argv[1] = 0;
```
```
  pipe(p);
  if(fork() == 0) {
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    exec("/bin/wc", argv);
  } else {
    write(p[1], "hello world\n", 12);
    close(p[0]);
    close(p[1]);
  }
```
The program calls pipe, which creates a new pipe and records the read and write file descriptors in the array p. After fork, both parent and child have file descriptors referring to the pipe. The child dups the read end onto file descriptor 0, closes the file descriptors in p, and execs wc. When wc reads from its standard input, it reads from the pipe. The parent writes to the write end of the pipe and then closes both of its file descriptors.

这里定义了两个整型变量`p[0]`和`p[1]`用于`pipe`调用，而`pipe`的作用就是将这两个变量作为文件描述将其联系起来，其中一个（`p[0]`）作为输入也就是读端的文件描述符，另一个（`p[1]`）作为输出也就是写端的文件描述符：

> The `pipe()` function shall create a pipe and place two file descriptors, one each into the arguments `fildes[0]` and `fildes[1]`, that refer to the open file descriptions for the read and write ends of the pipe. 

在`pipe`调用之后，所有输出到`p[1]`的内容都可以有`p[0]`读出，因此这里需要使用和实现重定向命令类似的方法来将标准输入（`stdin`，默认文件描述符为`0`）和标准输出（`stdout`，默认文件描述为`1`）重新绑定到`p[0]`和`p[1]`上。与重定向命令不同的是，这里并没有打开或创建一个实际的文件，而只是复用了文件描述符，所以这里使用了[`dup`](http://man7.org/linux/man-pages/man2/dup.2.html)调用来实现这个功能：

> The `dup()` system call creates a copy of the file descriptor oldfd, using the lowest-numbered unused file descriptor for the new descriptor.

实现重新绑定标准输出的文件描述符（`1`）的流程为：

1. 使用`close(1)`关闭并回收标准输出的文件描述符（`1`），使其对后续的`dup`可用，该函数执行完成之后，文件描述符`1`已经不在和默认的标准输出（通常是终端）相联系了
2. 使用`dup(p[1])`将文件描述符`1`复制为和`p[1]`的拷贝，此时，对`1`的写操作（`write`）的（也就是输出到标准输出）都和对`p[1]`的写操作相同
3. 使用`close(p[0]`和`close(p[1])`关闭文件描述符`p[0]`和`p[1]`，这里需要注意到`pipe`的[两个特性](http://man7.org/linux/man-pages/man7/pipe.7.html)：

    1. 如果一个进程尝试从一个空的pipe（empty pipe）中读取数据，`read`调用将导致该进程被block，如果一个进程尝试向一个满的pipe中（full pipe）中写数据，`write`调用将导致该进程被block：
    
> If a process attempts to read from an empty pipe, then `read(2)` will block until data is available.  If a process attempts to write to a full pipe (see below), then `write(2)` blocks until sufficient data has been read from the pipe to allow the write to complete.

    2. 如果所有引用
    
> If all file descriptors referring to the write end of a pipe have been closed, then an attempt to read(2) from the pipe will see end-of-file (read(2) will return 0).  If all file descriptors referring to the read end of a pipe have been closed, then a write(2) will cause a SIGPIPE signal to be generated for the calling process.

**[TBD]**

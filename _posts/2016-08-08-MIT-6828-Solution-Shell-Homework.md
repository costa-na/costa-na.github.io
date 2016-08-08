---
layout: post
title: MIT 6.828课程解析（五） Lecture 1：sh.c功能补全
---

## 简介
补全[sh.c](https://pdos.csail.mit.edu/6.828/2014/homework/sh.c)缺失的功能是6.828第一个[homework](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)，其目标是通过实现shell的部分功能让学生熟悉Unix的系统调用接口和shell的使用。

我们在上一篇（[MIT 6.828课程解析（四） Lecture 1：sh.c分析](https://costa-na.github.io/2016/08/07/MIT-6828-Solution-Simplified-Shell/)）已经详细分析了sh.c的结构和逻辑，在分析的最后，我们可以看到`runcmd`这个函数缺少对三种不同命令的处理逻辑，因此，我们在这篇文章中将会补全这些缺失的逻辑代码，让sh.c可以完整的编译运行起来。

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

因此，在这个`case`分支，我们只需要调用`exec`类系统调用，将欲执行的命令字符串和其参数数组传递给它，就可以完成命令的执行了。这里需要注意的是，由于sh.c是编译在Linux环境下面的，因此这里需要从[`exec`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/exec.html)家族函数中选一个合适的函数来执行此功能，`exec`类系统调用一共有六种：`execl`、`execv`、`execle`、`execve`、`execlp`和`execvp`，因为我们要传递给其的是可执行文件的具体路径（`path`）和一个`char`指针数组（`argv`），所以这里选择了`execv`，补全后的代码如下：

```
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);
    if(execv(ecmd->argv[0], ecmd->argv) < 0){ // call execv to execute specified command
      perror("execv");                        // print error message if execution is failed
      exit(0);                                // terminate process
    }
    break;
```

在上篇我们已经说过，在`execcmd`的`argv`数组中存放的是命令的调用参数，其第一个元素存放的是命令自身的可执行文件的路径，所以这里我们将`ecmd->argv[0]`作为第一个参数传递给了`execv`。我们需要判断`execv`是否执行成功，如果执行失败了，就使用[perror](http://pubs.opengroup.org/onlinepubs/9699919799/functions/perror.html)打印错误信息，然后使用`exit(0)`结束进程。

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

这里需要知道[`close`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/close.html)的作用是关闭并回收一个文件描述符（`fd`），让其能够被之后的`open`或其他文件操作函数使用：

> The `close()` function shall deallocate the file descriptor indicated by `fildes`. To deallocate means to make the file descriptor available for return by subsequent calls to `open()` or other functions that allocate file descriptors. 

然后还要知道[`open`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html)的作用是分配一个文件描述符（`fd`），并将其和指定的文件相关联。`open`分配的文件描述符是文件描述符表中最小的可用文件描述符：

> The `open()` function shall return a file descriptor for the named file that is the lowest file descriptor not currently open for that process. 

因为要重定向的文件描述符已经存在于`rcmd->fd`中，所以这里我们先使用`close(rcmd->fd)`关闭需要重定向的文件描述符，再使用`open(rcmd->file, ...)`打开。`open`的第二个参数指定打开文件的`mode`，这个值已经存放在`rcmd->mode`中了，直接传递给`open`即可。当指定打开的文件不存在时，`open`会新建一个文件然后再尝试打开，其第三个参数即指定了新建文件的属性，这里我们使用了[`<sys/stat.h>`](http://pubs.opengroup.org/onlinepubs/9699919799/xsh/sysstat.h.html)中的两个常量来指定：

* `S_IRWXU`：ower具有读写和可执行权（read, write, execute/search by owner）
* `S_IRGRP`：group具有可读权（read permission, group）
* 
因此，补全后的代码如下：

```
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    close(rcmd->fd);                                        // close file descriotion '0' or '1'
    if(open(rcmd->file, rcmd->mode, S_IRWXU|S_IRGRP) < 0){  // opeon specified file, make fd '0' or '1' to refer to it 
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

这里定义了两个整型变量`p[0]`和`p[1]`用于`pipe`调用，而`pipe`的作用就是将这两个变量作为pipe两端的文件描述将其联系起来，其中一个（`p[0]`）作为输入也就是读端的文件描述符，另一个（`p[1]`）作为输出也就是写端的文件描述符：

> The `pipe()` function shall create a pipe and place two file descriptors, one each into the arguments `fildes[0]` and `fildes[1]`, that refer to the open file descriptions for the read and write ends of the pipe. 

在`pipe`调用之后，所有输出到`p[1]`的内容都可以由`p[0]`读出，因此这里需要使用与实现重定向命令类似的方法来将标准输入（`stdin`，默认文件描述符为`0`）和标准输出（`stdout`，默认文件描述为`1`）重新绑定到`p[0]`和`p[1]`上。与重定向命令不同的是，这里并没有打开或创建一个实际的文件，而只是复用了文件描述符，所以这里使用了[`dup`](http://man7.org/linux/man-pages/man2/dup.2.html)调用来实现这个功能：

> The `dup()` system call creates a copy of the file descriptor oldfd, using the lowest-numbered unused file descriptor for the new descriptor.

对pipe命令的左子命令（`pcmd->left`）和右子命令（`pcmd->right`）是通过分别调用`fork1`来创建新的子进程，并在新创建的子进程中递归调用`runcmd`来执行的。由于`fork1`的实质是对系统调用[`fork`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html)的封装，而由`fork`产生的子进程将会和父进程共享所有的父进程中的内存数据，包括了在父进程中定义的变量（`p[0]`和`p[1]`），打开的文件描述符（`0`、`1`）。因此，对标准输入的文件描述符（`0`）和标准输出的文件描述符（`1`）的重新绑定的工作都是在新创建的子进程中来完成的。由于pipe是让左子命令执行产生的输出，作为右子命令执行的输入，因此需要在执行左子命令的子进程中重新绑定标准输出的文件描述符（`1`），在执行右子命令的子进程中重新绑定标准输入的文件描述符（`0`）。

在新创建的用于执行左子命令的子进程中实现重新绑定标准输出的文件描述符（`1`）并执行子命令的流程为：

1. 使用`close(1)`关闭并回收标准输出的文件描述符（`1`），使其对后续的`dup`可用，该函数执行完成之后，文件描述符`1`已经不再和默认的标准输出设备（通常是终端）相联系了
2. 使用`dup(p[1])`将文件描述符`1`复制为和`p[1]`的拷贝，此时，对`1`的写操作（`write`）的（也就是输出到标准输出）都和对`p[1]`的写操作相同
3. 使用`close(p[0]`和`close(p[1])`关闭文件描述符`p[0]`和`p[1]`，这里需要注意到`pipe`的[两个特性](http://man7.org/linux/man-pages/man7/pipe.7.html)：1. 如果一个进程尝试从一个空的pipe（empty pipe）中读取数据，`read`调用将导致该进程被block；如果一个进程尝试向一个满的pipe中（full pipe）中写数据，`write`调用将导致该进程被block。 2. 如果所有与pipe的写端相关联的文件描述符都被关闭了的话，对pipe的读操作将会读到`EOF`；如果所有与pipe的读端相关联的文件描述符都被关闭了的话，对pipe的写操作将会收到由调用写操作的进程产生的`SIGPIPE`信号。因为此时我们只需要将pipe写端的文件描述符`p[1]`和标准输出（`1`）相关联，而`p[0]`是用于在执行右子命令的子进程中和标准输入（`0`）相关联的，在当前子进程中我们并不需要`p[0]`，所以这里使用`close(p[0])`将其关闭。而如果在当前子进程中执行完`dup(p[1])`之后，不关闭pipe的写端文件描述`p[1]`的话，执行右子命令的子进程将pipe中的数据读取完毕pipe为空之后，由于还存在对pipe写端的引用（`p[1]`），导致其将会block在pipe上，无法继续执行下去，所以这里也需要使用`close(p[1])`将pipe的写端关闭。
4. 递归调用`runcmd`来执行pipe命令的左子命令（`pcmd->left`），因为左子命令可以是可以执行命令（`execcmd`）或者重定向命令（`redircmd`），所以这里不能单独使用`execv`，而必须再次调用`runcmd`。

用于执行右子命令的子进程中实现重新绑定标准输入的文件描述符（`0`）并执行子命令的流程与以上描述的步骤绝大部分相同。其不同之处在于：1. 对标准输入文件描述符的拷贝需要先使用`close(0)`，关闭标准输入文件描述符，再使用`dup(p[0])`，将标准输入绑定到pipe的读端（`p[0]`）；2. 对右子命令的执行是通过使用`pcmd->right`作为参数调用`runcmd`来实现的。

现在用于执行子命令的子进程都已经创建好了，回到父进程也就是shell中，由于父进程和子进程共享相同的文件描述符，在父进程中也需要使用`close(p[0])`和`close(p[1])`来分别关闭对pipe读端和写端的引用，使得子进程不会block在空的pipe上。最后，父进程通过连续调用两次[`wait`](http://pubs.opengroup.org/onlinepubs/9699919799/functions/wait.html)，来分别等待两个子进程执行结束，因此，补全后的代码如下：

```
  case '|':
    pcmd = (struct pipecmd*)cmd;
    pipe(p);                // call pipe to create two fd (p[0] and p[1]) to refer to read-end and write-end of pipe separately
    if(fork1() == 0){       // call fork to create subprocess to execute left-subcommand
      close(1);             // close fd to stdout('1')
      dup(p[1]);            // make '1' and p[1] both refer to write-end of pipe
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->left);   // call runcmd to run left-subcommand
    }

    if(fork1() == 0){       // call fork to create subprocess to execute right-subcommand
      close(0);             // close fd to stdin('0')
      dup(p[0]);            // make '0' and p[0] both refer to read-end of pipe
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->right);  // call runcmd to run left-subcommand
    }

    close(p[0]);
    close(p[1]);
    wait();
    wait();
    break;
```

## 总结
至此，sh.c的基本功能已经完成，我们可以编译sh.c并运行，其不仅支持单条可执行的命令（因为还未实现对全局变量`$PATH`的支持，所以输入的命令必须是可执行文件的完整路径，如：`/bin/ls`），而且支持重定向和管道，我们将在后续分析xv6时，还会看到一个功能更为全面的shell。

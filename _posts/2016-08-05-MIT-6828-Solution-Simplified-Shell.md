---
layout: post
title: MIT 6.828课程解析（四） Lecture 1：sh.c分析
---

## 简介
补全[sh.c](https://pdos.csail.mit.edu/6.828/2014/homework/sh.c)缺失的功能是6.828第一个[homework](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)，其目标是通过实现shell的部分功能让学生熟悉Unix的系统调用接口和shell的使用。

sh.c这个文件实现了一个简化版的shell，其默认提供了大部分框架代码，需要手动补充的缺失的功能是：
* 实现单条命令的执行
* 实现重定向（redirect）
* 实现管道（pipe）

在动手写代码之前，先让我们浏览一遍整个sh.c的框架，力图理解其总体的逻辑结构。


## 分析
sh.c只是一个普通的c代码文件，其入口函数和其他c代码一样，都是`main`函数，所以让我们先从`main`函数开始，作为整个分析的入口点

### `main`
![main](/public/img/main_code.png)

sh.c的`main`函数实现非常简单，在创建了一个命令读取的缓存之后，就进入一个“读命令” > “执行命令” > “读下一条命令” > “执行” > ...的循环中，直到读取命令返回失败（读到`EOF`），然后就调用`exit(0)`结束自身进程。其流程图如下：

![main](/public/img/main.png)

要点：

1. 命令缓存是由`main`在静态分配内存中分配的一个100字节的char型数组（[What does “static” mean in a C program?](http://stackoverflow.com/questions/572547/what-does-static-mean-in-a-c-program)）
2. 如果读取命令成功，`main`会在调用`fork1`之前判断用户输入的命令是否为cd。为何要在父进程（`main`自身）而不是在执行命令的子进程中来判断，是因为cd命令会改变当前进程的工作目录（[Working directory](https://en.wikipedia.org/wiki/Working_directory)），从而改变后续命令的执行环境，所以需要在父进程中处理cd命令
3. `main`对cd处理得不是很好，这里存在一个隐藏的bug，如果在用户输入的命令之后存在空格，如“cd /bin  ”，[`chdir`](http://pubs.opengroup.org/onlinepubs/009695399/functions/chdir.html)将执行失败，因此这里需要将命令结尾处的空白字符strip掉
4. `main`在调用`fork1`之后会使用[`wait`](http://pubs.opengroup.org/onlinepubs/009695399/functions/wait.html)来等待子进程执行结束，所以改变此处逻辑，让`main`分发下子进程后，不等待立即返回，可以使sh.c支持后台执行（这个是该homework的challenge之一）
5. `main`并没有直接调用`fgets`来获取用户输入，而是将`fgets`封装在了`getcmd`中，这样可以在读取用户输入之前，更改输出的提示符

### `getcmd`
![main](/public/img/getcmd_code.png)

`getcmd`只是简单封装了`fgets`，其主要功能是：

1. 判断[`stdin`](http://pubs.opengroup.org/onlinepubs/009695399/functions/stdin.html)是否为终端设备（terminal），如果是，就输出提示符“6.828$ ”。想要将shell的提示符更改为其他形式就可以修改此处
2. 使用[`memset`](http://pubs.opengroup.org/onlinepubs/009695399/functions/memset.html)清空命令缓存数组`buf`
3. 调用[`fgets`](http://pubs.opengroup.org/onlinepubs/009695399/functions/fgets.html)从`stdin`读取一行用户输入（以回车符`\n`结束），将其存到`buf`中
  * **注意**：`fgets`读取的最大字符数比第二个参数指定个数的要少一个，这是因为要在最后一个位置上填上`NULL`表示字符串的结束
4. 判断读取是否成功，成功返回0，读取失败（读到`EOF`）返回-1，这个是Unix的惯用法（成功返回0，失败返回-1）

### `fork1`
`fork1`封装了[`fork`](http://pubs.opengroup.org/onlinepubs/009695399/functions/fork.html)，这样在`fork`执行失败（返回-1）的时候，可以调用[`perror`](http://pubs.opengroup.org/onlinepubs/009695399/functions/perror.html)打印错误信息，便于使用和调试

至此`main`的主体逻辑已经分析完毕，接下来分析命令的解析和具体执行逻辑，这部分占了sh.c的大部分代码。

### `parsecmd`预览
`parsecmd`从名字上可以推断，该函数主要执行对用户输入的命令字符串的解析，`parsecmd`主体逻辑并不复杂，但此时我们只需要概括的看一下它的执行流程，并不会深入到每个子函数中去。

1. 通过[`strlen`](http://pubs.opengroup.org/onlinepubs/009695399/functions/strlen.html)定位到命令字符串结尾后面的第一个位置上
2. 将字符串的起始位置和结尾后面的第一个位置传给`parseline`，返回值赋给`struct cmd`类型的变量`cmd`
3. 再将相同的参数连同一个空字串传给`peek`函数
4. 判断命令行是否已经解析完毕，如果还有剩余的字符，打印错误信息，结束进程
5. 将`cmd`返回给`runcmd`

到这里我们大致了解了`parsecmd`在做什么：解析命令字符串，将其转换为一个`cmd`结构，然后交给`runcmd`执行。现在暂时停止对各子函数的功能的深入分析，先让我们了解一下几个工具函数的实现，这样在后面遇到的时候就不用再去查看了。

### `peek`
`peek`函数的第一个参数是一个二重char指针（`char **`），而我们是将一个指向命令字符串起始位置的指针的**地址**（`&s`），作为第一个参数传给`peek`，因此该参数即是输入参数，也是输出参数，在`peek`处理完成之后，`s`应指向字符串其他位置。
`peek`函数的第二个参数可以参考`parsecmd`中对`peek`的调用。在`parsecmd`中，调用`peek`的第二个参数是指向`s`指向的字符串结尾后面的第一个位置的指针。

因此，`peek`的作用如下：

1. 使用`while`循环去掉命令前面的空白字符，这里的空白字符包括`' '`、`'\t'`、`'\r'`、`'\n'`和`'\v'`，这里使用了一个字符串函数[`strchr`](http://pubs.opengroup.org/onlinepubs/009695399/functions/strchr.html)来简化了判断代码
2. 判断当前字符串起始字符是否是于输入的字符串（第三个参数，`toks`）其中任何一个字符，比如输入`"<>"`，判断该字符是否为`'<'`或`'>'`。这里用于判断该命令的类型（是单独的命令，还是redirect，还是pipe）

总结一下`peek`的功能：消除空白符，判断命令类型

### `gettoken`
该函数较为复杂，我们一步一步来看。
首先，先来看返回值`ret`。函数中对`ret`赋值的地方有两处`ret = *s`和`ret = 'a'`，注意观察到在`ret = *s`之后有一个`switch`逻辑：

```
    ret = *s;
    switch (*s) {
        case 0:
            break;
        case '|':
        case '<':
            s++;
            break;
        case '>':
            s++;
            break;
        default:
            ret = 'a';
            while (s < es && !strchr(whitespace, *s) && !strchr(symbols, *s))
            s++;
            break;
    }
```

因此，`ret`的值只能是固定的几种，要么是`'0'`、`'|'`、`'<'`、`'>'`，要么是`'a'`，而`ret`的值是根据`*s`的值来决定的，可以概括如下：如果`s`指向一个普通字符（除了`'0'`、`'|'`、`'<'`、`'>'`之外的任何字符），`gettoken`返回`'a'`，否则返回指向的这个特殊字符。

然后，再让我们来看看`gettoken`被调用的情况。对`gettoken`调用存在于四个地方：

1. `parseline`：`gettoken(ps, es, 0, 0);`
2. `parseredirs`：`tok = gettoken(ps, es, 0, 0);`
3. `parseredirs`：`if(gettoken(ps, es, &q, &eq) != 'a')`
4. `parseexec`：`if((tok=gettoken(ps, es, &q, &eq)) == 0)`

为了简化分析，先让我们来看看当传给`gettoken`的最后两个参数为0时的情况。
在`parseredirs`中，对`gettoken`的第一次调用是`while(peek(ps, es, "<>"))`循环中的第一条语句，因此我们可以猜测，传给`gettoken`的前两个参数的意义和传给`peek`的意义相同，都是表示一个字符串的起始字符和结束字符后第一个空位，并且第一个参数都是输入/输出参数。

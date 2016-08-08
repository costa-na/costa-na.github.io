---
layout: post
title: MIT 6.828课程解析（四） Lecture 1：sh.c分析
---

## 简介
补全[sh.c](https://pdos.csail.mit.edu/6.828/2014/homework/sh.c)缺失的功能是6.828第一个[homework](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)，其目标是通过实现shell的部分功能让学生熟悉Unix的系统调用接口和shell的使用。

sh.c这个文件实现了一个简化版的shell，其默认提供了大部分框架代码，需要手动补充的缺失的功能是：

* 实现单条命令的执行
* 实现重定向（[redirection](https://www.wikiwand.com/en/Redirection_(computing))）
* 实现管道（[pipe](https://www.wikiwand.com/en/Pipeline_(Unix))）

在动手写代码之前，先让我们浏览一遍整个sh.c的框架，力图理解其总体结构。

## 数据结构定义

### 命令结构类型定义
sh.c的功能是解析用户输入的命令并执行，由于命令类型可以分为：

* 独立命令，如： `ls`，`echo "Hello, world!"`
* 重定向命令，如：`ls > list`，`echo < in_file > out_file`
* 管道命令，如：`ls | sort | cat`

为了将用户输入的命令转换为一种有序结构，以便于解析和执行，sh.c采用了一种常用的方法来模拟面向对象编程语言中的继承属性。

在C语言标准（[ISO/IEC 9899:201x](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)）中，关于指向结构的指针是这样描述的：

> 6.7.2.1.15 Within a structure object, the non-bit-field members and the units in which bit-fields
reside have addresses that increase in the order in which they are declared. **A pointer to a
structure object, suitably converted, points to its initial member (or if that member is a
bit-field, then to the unit in which it resides), and vice versa.** There may be unnamed
padding within a structure object, but not at its beginning.

“指向结构体的指针可以转换为指向结构体内第一个成员变量的指针，反之亦然”。因此，我们可以定义不同的结构，只要保证其第一个成员变量类型一致，就可以在指向这几种的结构体的指针任意转换，这样就可以在一定程度上模拟面向对象的一些特性。sh.c就利用了此方法，其中定义了四种不同的cmd类型的结构，分别用以代表上面提到的三种命令类型，用`type`字段的不同值来作区分：

* `struct cmd`，不代表实际命令，其作用相当于Java中的抽象基类
* `struct execcmd`，代表独立命令，`type`字段为`' '`
* `struct redircmd`，代表重定向命令，`type`字段为`'<'`或`'>'`
* `struct pipecmd`，代表管道类型，`type`字段为`'|'`

这里需要注意到，在`execcmd`的定义中，只有`type`类型字段和`argv`参数字符串数组，并没有存放命令的字段，这是因为在Unix惯例中，参数数组的第1个元素（`argv[0]`）即是需执行的命令的字符串，如执行`ls /bin`这个命令，shell调用ls时，传递给ls的参数数组`argv`的第一个元素就是字符串`"ls"`。所以并不需要独立的字段来存放具体的命令。

由于这四种类型的第一个字段都是`int type`，利用上面提到的C语言中指向结构的指针的特性，就可以在指向这四种结构体的指针之间作任意的转换。

### `execcmd`
![execcmd](/public/img/6.828/homework/1/execcmd_func_code.png)

`execcmd`函数分配一个`execcmd`类型的结构，将其内存布局初始化为`0`，填写其`type`字段，然后将其地址转换为`cmd`指针并返回。

### `redircmd`
![redircmd](/public/img/6.828/homework/1/redircmd_func_code.png)

`redircmd`函数分配一个`redircmd`类型的结构，将其内存布局初始化为`0`，根据输入的参数填写结构对应的字段，如：类型（`type`）、需要运行的命令（`cmd`）、输入/输出文件（`file`）、打开模式（`mode`）和打开的文件描述符（`fd`），然后将其地址转换为`cmd`指针并返回。

### `pipecmd`
![pipecmd](/public/img/6.828/homework/1/pipecmd_func_code.png)

`redircmd`函数分配一个`redircmd`类型的结构，将其内存布局初始化为`0`，根据输入的参数填写结构对应的字段：类型（`type`）、`'|'`左边的子命令（`left`）和`'|'`右边的子命令（`right`），然后将其地址转换为`cmd`指针并返回。

### `mkcopy`
![mkcopy](/public/img/6.828/homework/1/mkcopy_code.png)

这个函数很简单，就只是将输入的由`s`和`es`指定的字符串复制了一份，在末尾加上了字符串结尾的`0`，然后就将其返回了。

## 函数
sh.c只是一个普通的c代码文件，其入口函数和其他c代码一样，都是`main`函数，所以让我们先从`main`函数开始，作为整个分析的入口点

### `main`
![main](/public/img/6.828/homework/1/main_code.png)

sh.c的`main`函数实现非常简单，在创建了一个命令读取的缓存之后，就进入一个“读命令” > “执行命令” > “读下一条命令” > “执行” > ...的循环中，直到读取命令返回失败（读到[`EOF`](https://en.wikipedia.org/wiki/End-of-file)），然后就调用[`exit(0)`](http://pubs.opengroup.org/onlinepubs/009695399/functions/exit.html)结束自身进程。其流程图如下：

![main](/public/img/6.828/homework/1/main.png)

要点：

1. 命令缓存是由`main`在静态分配内存中分配的一个100字节的char型数组（[What does “static” mean in a C program?](http://stackoverflow.com/questions/572547/what-does-static-mean-in-a-c-program)）
2. 如果读取命令成功，`main`会在调用`fork1`之前判断用户输入的命令是否为[cd](http://pubs.opengroup.org/onlinepubs/009696699/utilities/cd.html)。为何要在父进程（`main`自身）而不是在执行命令的子进程中来判断，是因为cd命令会改变当前进程的工作目录（[Working directory](https://en.wikipedia.org/wiki/Working_directory)），从而改变后续命令的执行环境，所以需要在父进程中处理cd命令
3. `main`对cd处理得不是很好，这里存在一个隐藏的bug，如果在用户输入的命令之后存在空格，如“cd /bin  ”，[`chdir`](http://pubs.opengroup.org/onlinepubs/009695399/functions/chdir.html)将执行失败，因此这里需要将命令结尾处的空白字符strip掉
4. `main`在调用`fork1`之后会使用[`wait`](http://pubs.opengroup.org/onlinepubs/009695399/functions/wait.html)来等待子进程执行结束，所以改变此处逻辑，让`main`分发下子进程后，不等待立即返回，可以使sh.c支持后台执行（这个是该homework的challenge之一）
5. `main`并没有直接调用`fgets`来获取用户输入，而是将`fgets`封装在了`getcmd`中，这样可以在读取用户输入之前，更改输出的提示符

### `getcmd`
![main](/public/img/6.828/homework/1/getcmd_code.png)

`getcmd`只是简单封装了`fgets`，其主要功能是：

1. 判断[`stdin`](http://pubs.opengroup.org/onlinepubs/009695399/functions/stdin.html)是否为终端设备（terminal），如果是，就输出提示符“6.828$ ”。想要将shell的提示符更改为其他形式就可以修改此处
2. 使用[`memset`](http://pubs.opengroup.org/onlinepubs/009695399/functions/memset.html)清空命令缓存数组`buf`
3. 调用[`fgets`](http://pubs.opengroup.org/onlinepubs/009695399/functions/fgets.html)从`stdin`读取一行用户输入（以回车符`\n`结束），将其存到`buf`中
  * **注意**：`fgets`读取的最大字符数比第二个参数指定个数的要少一个，这是因为要在最后一个位置上填上`NULL`表示字符串的结束
4. 判断读取是否成功，成功返回0，读取失败（读到`EOF`）返回-1，这个是Unix的惯用法（成功返回0，失败返回-1）

### `fork1`
![main](/public/img/6.828/homework/1/fork1_code.png)

`fork1`封装了[`fork`](http://pubs.opengroup.org/onlinepubs/009695399/functions/fork.html)，这样在`fork`执行失败（返回-1）的时候，可以调用[`perror`](http://pubs.opengroup.org/onlinepubs/009695399/functions/perror.html)打印错误信息，便于使用和调试

至此`main`的主体逻辑已经分析完毕，接下来分析命令的解析和具体执行逻辑，这部分占了sh.c的大部分代码。

### `parsecmd`分析 1
![parsecmd](/public/img/6.828/homework/1/parsecmd_code.png)

`parsecmd`从名字上可以推断，该函数主要执行对用户输入的命令字符串的解析，`parsecmd`主体逻辑并不复杂，但此时我们只需要概括的看一下它的执行流程，并不会深入到每个子函数中去。

1. 通过[`strlen`](http://pubs.opengroup.org/onlinepubs/009695399/functions/strlen.html)定位到命令字符串结尾后面的第一个位置上
2. 将字符串的起始位置和结尾后面的第一个位置传给`parseline`，返回值赋给`struct cmd`类型的变量`cmd`
3. 再将相同的参数连同一个空字串传给`peek`函数
4. 判断命令行是否已经解析完毕，如果还有剩余的字符，打印错误信息，结束进程
5. 将`cmd`返回给`runcmd`

到这里我们大致了解了`parsecmd`在做什么：解析命令字符串，将其转换为一个`cmd`结构，然后交给`runcmd`执行。现在先让我们了解一下几个工具函数的实现，这样在后面遇到的时候就不用再去查看了。

### `peek`
![peek](/public/img/6.828/homework/1/peek_code.png)

`peek`函数的第一个参数是一个二重char指针（`char **`），而我们是将一个指向命令字符串起始位置的指针的**地址**（`&s`），作为第一个参数传给`peek`，因此该参数即是输入参数，也是输出参数，在`peek`处理完成之后，`s`应指向字符串其他位置。
`peek`函数的第二个参数可以参考`parsecmd`中对`peek`的调用。在`parsecmd`中，调用`peek`的第二个参数是指向`s`指向的字符串结尾后面的第一个位置的指针。

因此，`peek`的作用如下：

1. 使用`while`循环去掉命令前面的空白字符，这里的空白字符包括`' '`、`'\t'`、`'\r'`、`'\n'`和`'\v'`，这里使用了一个字符串函数[`strchr`](http://pubs.opengroup.org/onlinepubs/009695399/functions/strchr.html)来简化了判断代码
2. 判断当前字符串起始字符是否是于输入的字符串（第三个参数，`toks`）其中任何一个字符，比如输入`"<>"`，判断该字符是否为`'<'`或`'>'`。这里用于判断该命令的类型（是单独的命令，还是redirect，还是pipe）

总结一下`peek`的功能：消除空白符，判断命令类型。

### `gettoken`
![gettoken](/public/img/6.828/homework/1/gettoken_code.png)

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

因此，`ret`的值只能是固定的几种，要么是`'0'`、`'|'`、`'<'`、`'>'`，要么是`'a'`，而`ret`的值是根据`*s`的值来决定的，可以概括如下：如果`s`指向一个普通字符（因为之前已经使用`while`循环去掉了空白符，所以这里的普通字符应该是除了空白符、`'0'`、`'|'`、`'<'`、`'>'`之外的任何字符），`gettoken`返回`'a'`，否则返回指向的这个特殊字符。

当第三个参数和第四个参数为`0`时，对该函数的调用`gettoken(ps, es, 0, 0)`，其运行的流程为：

1. 跳过字符串开头的所有空白符
2. 使用`switch`判断当前`s`指向的字符类型
   1. 如果是`0`（空字符），不作处理，跳至3
   2. 如果是特殊字符（`'|'`、`'<'`、`'>'`其中一种），将`s`指向后一个字符后，跳至3
   3. 如果是普通字符，表示其是一个token的开始，将返回值`ret`设置为`'a'`，使用一个`while`循环将`s`指向以该token的结尾后一个位置。`while`循环中的判断条件是：`s`未到达整个输入字符串的结尾处（`s < es`），并且`*s`不是空白符 ，并且`*s`不是特殊字符，跳至3
3. 再次跳过到下一个非空字符前面的空白符
4. 返回`ret`

现在可以来分析如果调用`gettoken`第三个参数和第四个参数不为`0`时的情况了。在处理了输入字符串开头的空白符后，`gettoken`将`*q`指向了第一个非空白符的字符，在`switch`中，当`*s`是特殊字符时，`s`向后移动一位，然后跳出`switch`，接着将`s`赋值给`*eq`，因此现在`*q`指向一个特殊字符，而`*eq`指向该特殊字符后面的第一个位置上，由此可见，`*q`和`*eq`恰好指定了一个只含有一个特殊字符的字符串。

同理，回想一下前述流程中的2.3，当`*s`是普通字符时，`switch`中`default`分支的`while`循环会将`s`移动到以普通字符开头的字符串的结尾处的下一个位置上，然后跳出`switch`，将`s`赋值给`*eq`，此时`*q`和`*eq`指定了一个不包含空白符和特殊字符的字符串。

至此，`gettoken`分析完毕。
总结一下，`gettoken`判断下一个非空字符的类型，如果下一个是`'|'`、`'<'`、`'>'`其中一种，或者是一个不包含空白符和特殊字符的字符串。

现在我们可以回到`parsecmd`，继续分析其调用的各个子函数了。

### `parseline`
![parseline](/public/img/6.828/homework/1/parseline_code_new.png)

`parseline`只有三条语句，仅仅是调用了`parsepipe`，然后将`parsepipe`的返回值再返回给调用者。

### `parsepipe`分析 1
![parsepipe](/public/img/6.828/homework/1/parsepipe_code_new.png)

从名字上看，`parsepipe`主要作用是解析pipe类型的命令，比如`ls | cat`，或者`ls | sort | cat`。其第一条语句调用了`parseexec`，所以这里我们先放下`parsepipe`，先来看看`parseexec`做了什么。

### `parseexec`
![parseexec](/public/img/6.828/homework/1/parseexec_code.png)

`parseexec`声明了一对`char`指针`q`和`eq`，用于获取从命令行解析到的token，声明了一个`execcmd`指针`cmd`，用于指向分配的`execcmd`结构，还声明了一个`cmd`指针`ret`，用于返回。该函数具体执行流程为：

1. 调用`execcmd()`分配一个`execcmd`结构，用于存放独立命令和其参数
2. 将代表命令参数个数的变量`argc`初始化为0
3. 调用`parseredirs`，解析重定向命令。为何要在解析独立命令之前就先解析redir命令，是因为Unix的shell支持直接在命令行上使用`< file`或者`> file`这样的命令将`file`中的内容输出到`stdout`，或将`stdin`得到的输入输出到`file`。
4. 循环调用`peek`，确定只解析pipe符（`'|'`）之前的命令（如果是pipe命令的话）
5. 在循环中，先调用`gettoken`获取到一个token
6. 如果得到的token是空字符（`gettoken`返回0），跳出循环
7. 如果得到的token不是普通字符（`gettoken`返回不是`'a'`），打印错误信息，结束进程。这是为了处理如`$ | cat`这样的错误语法
8. 调用`mkcopy`为获取到的token分配存储空间，然后使用`argv[argc]`将当前`mkcopy`返回的地址放进`argv`中空闲的槽中，移动`argc`使其指向`argv`下一个空闲的槽
9. 判断`argc`是否超过设定的最大值，如果超过，打印错误信息，结束进程
10. 调用`parseredirs`解析独立命令之后的redir命令
11. 结束循环，返回完成解析后的`execcmd`结构

`parseexec`完成了大部分的解析工作，其对redir命令的解析是通过`parseredirs`完成的，因为我们接下来需要看看`parseredirs`函数的流程。

### `parseredirs`
![parseredirs](/public/img/6.828/homework/1/parseredirs_code.png)

`parseredirs`使用了一个`while`循环来解析命令中出现的redir命令，其流程为：

1. 如果存在`<`或`>`字符，进入解析循环
2. 首先使用`gettoken`读入重定向符，并跳至重定向符后面一个token
3. 再次使用`gettoken`读入token，如果得到的不是普通字符，打印错误信息，结束进程，这是处理如`$ < < file`这样的错误语法
4. 根据获取到的重定向符，使用`redircmd()`构造不同的`redircmd`结构
5. 返回构造好的`redircmd`结构

至此，我们已经了解了sh.c是如何解析独立命令和redir命令了，我们可以返回到`parseline`，看看是如何解析pipe命令的

### `parsepipe`分析 2
只要弄懂了`parseexec`命令，对`parsepipe`的命令分析就变得非常简单，其流程如下：

1. 调用`parseexec`解析pipe字符（`'|'`）左边的命令（如果存在pipe字符的话）
2. 调用`peek`判断是否存在pipe字符
3. 如果存在，表明是pipe命令，先调用`gettoken`提过pipe字符，然后递归调用自身来解析pipe字符右边的命令
4. 将解析到的左边的子命令和右边的子命令传递给`pipecmd()`，构造一个`pipecmd`结构
5. 返回构造好的`pipecmd`结构

这里比较复杂的是第4步。首先我们需要看一下Unix是怎样来看待pipe命令的，对于简单的pipe命令，如:`ls | cat`，Unix会将其看作三部分：

1. 左子命令：`ls`
2. pipe字符：`'|'`
3. 右子命令：`cat`

这应该没有问题，那么Unix又是如何看到多条串联的pipe命令的呢，如：`ls | sort | uniq | cat`。Unix采取了一种递归的方式：

1. 左子命令：`ls`
2. pipe字符：`'|'`
3. 右子命令：`sort | uniq | cat`
    1. 左子命令：`sort`
    2. pipe字符：`'|'`
    3. 右子命令：`uniq | cat`
        1. 左子命令：`uniq`
        2. pipe字符：`'|'`
        3. 右子命令：`cat`
 
这里还可以用树的形式来表示：

![main](/public/img/6.828/homework/1/pipecmd.png)

所以，`parsepipe`使用了递归的方式来构造`pipecmd`结果，这也导致了在`runcmd`中采用了递归的形式来执行`pipecmd`

### `parsecmd`分析 2
之前对`pasrecmd`的结论做了初步的分析，现在再回过去看`parsecmd`，其流程就一目了然了：

1. 获取命令字符串尾
2. 将命令字符串传给`parseline`
3. 判断是否还有多余的字符，如果有，打印错误信息，结束进程
4. 返回到上层函数

现在我们已经完成了sh.c对命令解析的相关函数的分析，接下来我们来看看sh.c是如何执行的命令的，这只涉及到了一个函数`runcmd`。

### `runcmd`
![runcmd](/public/img/6.828/homework/1/runcmd_code.png)

`runcmd`主要由一个`switch`分支结构组成，其作用就是根据传入的`cmd`参数的`type`字段，针对不同类型的命令，采取不同的执行方式，这部分就是该Homework需要完成的，这部分的分析我们放在下一篇文章中来说明。


## 总结
至此，sh.c的分析工作已经完成，除了以上对每个函数的具体分析外，还有一些问题需要注意：

* 除了需要从`buf`上提取命令和参数构造`execcmd`结构外，对命令的所有的解析都是在`buf`上完成的，并且对`buf`的操作都是只读的，没有对`buf`作任何的修改
* `peek`的作用是“看”下一个字符，用来判断命令的类型，也可以用来消除多余的空白符；`gettoken`除了可以提取下一个token，也可以用来跳过相应的字符
* 命令的构造函数`execcmd`、`redircmd`和`pipecmd`，其返回类型都是`cmd`类型，而在`runcmd`中，根据`cmd`结构的第一个字段`type`的不同值，再将其还原为真实的类型
* 在`peek`和`gettoken`中都有消除多余空白符的逻辑
* 除了对于正常命令的解析外，还要包括对错误命令的判断和处理

从这不过300行的代码我们学到了很多东西，在完成6.828后我们还会继续学习编译原理相关的知识，到那时，就可以自己实现tokenizer和parser，就可以完成一个完整的shell的实现了。



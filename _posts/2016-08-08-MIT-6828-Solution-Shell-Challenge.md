---
layout: post
title: MIT 6.828课程解析（六） Lecture 1：sh.c挑战练习
---

## 简介
在上一篇（[MIT 6.828课程解析（五） Lecture 1：sh.c功能补全]https://costa-na.github.io/2016/08/08/MIT-6828-Solution-Shell-Homework/）中，我们已经完成了[Homework: Shell](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)指定的任务，补全了sh.c中缺失的功能，完成后的sh.c已经具备了对可执行命令、重定向和管道的支持，但相对目前流行的shell如bash，其仍然确实了非常多的高级功能，因此在Homework的最后，有三个额外的练习作为challenge提供给学生，让其为sh.c增加更多的功能。

**注意：因为此篇是challenge练习，所以目前并未全部完成，后续将会缓慢补全**

### 增加多条命令单行执行功能
当前流行的shell都支持多条命令写在同一行中，命令之间使用`';'`分割，这样shell就会从左至右依次执行。由于上篇中实现的sh.c已经具备了执行单条命令的功能，我们只需要增加对含有`';'`分隔符的命令行的解析，即可以支持多条命令单行执行功能。

**[TBD]**

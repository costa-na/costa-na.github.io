---
layout: post
title: MIT 6.828课程解析（一） 简介
---

**本文翻译自[MIT 6.828 overview](https://pdos.csail.mit.edu/6.828/2014/overview.html)**

> 6.828 teaches the fundamentals of engineering operating systems. You will study, in detail, virtual memory, kernel and user mode, system calls, threads, context switches, interrupts, interprocess communication, coordination of concurrent activities, and the interface between software and hardware. Most importantly, you will study the interactions between these concepts, and how to manage the complexity introduced by the interactions.

6.828讲授设计和实现操作系统的基础概念。你将会深入学习到以下概念：

* 虚拟内存（[virtual memory](https://en.wikipedia.org/wiki/Virtual_memory)）
* 内核态和用户态（[kernel and user mode](https://en.wikipedia.org/wiki/Protection_ring)）
* 系统调用（[system calls](https://en.wikipedia.org/wiki/System_call)）
* 线程（[threads](https://en.wikipedia.org/wiki/Thread_(computing))）
* 上下文切换（[context switches](https://en.wikipedia.org/wiki/Context_switch)）
* 中断（[interrupts](https://en.wikipedia.org/wiki/Interrupt)）
* 进程间通信（[interprocess communication](https://en.wikipedia.org/wiki/Inter-process_communication)）
* 并发的协调（coordination of concurrent activities）
* 软硬件接口（the interface between software and hardware）

最重要的是，你将学到这些概念之间是如何相互影响的，以及如何管理由这些影响引入的（系统）复杂性。

> To master the concepts, 6.828 is organized in three parts: lectures, readings, and a major lab. The lectures and readings familiarize you with the main concepts. The lab forces you to understand the concepts at a deep level, since you will build an operating system from the ground up. After the lab you will appreciate the meaning of design goals such "reducing complexity" and "conceptual integrity".

为了掌握这些概念，6.828由三部分组成：讲义（lectures）、阅读（readings）以及一个大型试验（a major lab）。讲义和阅读是为了让你熟悉主要的概念。试验迫使你深层次的理解这些概念，因为你将从零开始会构建一个操作系统。在试验之后，你将领会到诸如“降低复杂度”和“概念完整性”等设计目标的意义。

> The lectures are organized in two main blocks. The first block introduces one operating system, xv6 (x86 version 6), which is a re-implementation of Unix Version 6, which was developed in the 1970s. In each lecture we will take one part of xv6 and study its source code; homework assignments will help you prepare for these lectures. At the end of the first block (about half-way the term), you will understand the source code for one well-designed operating system for an Intel-based PC, which help you building your own operating system.

lecture由两个主要的部分组成。第一个部分介绍了一个操作系统，xv6（x86 version 6），一个开发于20世纪70年代的Unix version 6的重新实现。在每个lecture中，我们将会针对xv6的其中一个部分，学习它的源代码；布置的homwork将会辅助你准备这些lectures。在这部分的结尾（大于到一半的样子），你将会理解一个基于Intel PC的良好设计的操作系统的源代码，这将会帮助你构建你自己的操作系统。

> The second block of lectures covers important operating systems concepts invented after Unix v6. We will study the more modern concepts by reading research papers and discussing them in lecture. You will also implement some of these newer concepts in your operating system.

lecture的第二部分覆盖了在Unix v6之后出现的重要的操作系统概念。我们将会通过阅读研究论文和讨论学习更现代的概念。同时将会在自己的操作系统上实现这些新概念中的一部分。

> You may wonder why we are studying an operating system that resembles Unix v6 instead of the latest and greatest version of Linux, Windows, or BSD Unix. xv6 is big enough to illustrate the basic design and implementation ideas in operating systems. On the other hand, xv6 is far smaller than any modern production O/S, and correspondingly easier to understand. xv6 has a structure similar to many modern operating systems; once you've explored xv6 you will find much that is familiar inside kernels such as Linux.

你可能想知道为何我们要通过重新实现Unix v6而不是像Linux、Windows或者BSD Unix这些系统的最新版来学习操作系统。xv6已经足够阐述操作系统的设计和实现的基础概念。从另一方面来说，xv6远远小于任何一个现代的工业型操作系统，因此更容易理解。xv6具有类似于很多现代操作系统的结构，一旦你学习了xv6，你将在像Linux的（现代）内核中发现很多类似之处。

> The lab is the place where the rubber meets the road. In the lab you will internalize the details of the concepts and their interactions. For example, although you have seen virtual memory in 6.004, 6.033, and again in 6.828 lectures, you will soon discover, during the labs, that you didn't really understand virtual memory and how it interacts with other concepts.

该实验是理论联系实际的。在实验中，你将融合这些概念的细节和它们的相互关系。比如，虽然你已经在6.004、6.033以及6.828的lecture中看过了虚拟内存(virtual memory)内存的概念。你将会在实验中发现，你并没有真正理解虚拟内存，以及它是如何和其他概念相互影响的。

> The lab is split into 6 major parts that build on each other, culminating in a primitive operating system on which you can run simple commands through your own shell. We reserve the last lecture for you to demo your operating system to the rest of the class.

实验被分为6个主要部分，每个部分分开构建，到最后你会得到一个原生的操作系统，你可以在你自己写的shell里面运行简单的命令。我们遗留了最后的lecture，让你可以向班上其他的同学展示你的操作系统。

> The operating system you will build, called JOS, will have Unix-like functions (e.g., fork, exec), but is implemented in an exokernel style (i.e., the Unix functions are implemented mostly as user-level library instead of built-in to the kernel). The major parts of the JOS operating system are:
1. Booting
2. Memory management
3. User environments
4. Preemptive multitasking
5. File system, spawn, and shell
6. Network driver
7. Open-ended project

你构建的操作系统叫做JOS，其具有Unix-like功能（比如fork、exec），但是采用exokernel的方式实现的（绝大多数Unix功能实现为用户模式的库，而不是内置到内核中）。JOS操作系统的主要部分有：
1. 启动（Booting）
2. 内存管理（Memory management）
3. 用户环境（User environments）
4. 抢占式多任务（Preemptive multitasking）
5. 文件系统、spawn和shell（File system, spawn, and shell）
6. 网络驱动（Network driver）
7. 开放式项目（Open-ended project）

> We will provide skeleton code for pieces of JOS, but you will have to do all the hard work. You'll have design freedom for the details of the first few assignments, and freedom over the whole design in the last assignments. You'll find that xv6 helps you understand many of the goals you're trying to achieve in JOS, but that JOS occupies a very different point in the design and implementation space from xv6.

我们将会提供JOS的框架代码，但你必须做其他所有的工作。你将在前几个分配的任务和最后一个任务中自由地设计细节。在你尝试实现JOS的过程中，你将会发现xv6能帮助你理解许多的设计目标，但JOS具有与xv6非常不同的设计和实现方向。

> The first 6 assignments are done individually. The last assignment is a team assignment.

前6个任务需要独立完成，最后一个任务是需要团队协作。（Sure, I will try it by myself. :)）

> You will develop your JOS operating system for a standard x86-based personal computer, the same one used for xv6. To simplify development we will use a complete machine simulator (QEMU) in the class for development and debugging. This simulator is real enough, however, that you will be able to boot your own operating system on physical hardware if you wish.

你将在标准的基于x86体系的PC上实现你自己的操作系统，和xv6运行的一样。为了简化实现，我们将咋实现和调试的课程上使用一个完全的虚拟机（QEMU）。这个虚拟机仿真度很高，所以只要你愿意，你可以从真实的物理硬件上启动你的操作系统。

> At the end of the lab you will be able to find your way around the source code of most operating systems, and more generally, be comfortable with system software. You will understand many operating systems concepts in detail and will be able to use them in other environments. You will also understand the x86 processor and the C programming language well.

在实验结束时，你将会发现你能够在绝大多数的操作系统源代码中找到方向，更广义地说，可以更熟悉系统软件。你将会理解许多操作系统概念的细节，能够在其他环境使用它们。同时你将更好的理解x86处理器和C语言。



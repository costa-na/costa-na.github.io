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

lectures由两个主要的部分组成。第一个部分介绍了一个操作系统，xv6（x86 version 6），一个开发于20世纪70年代的Unix version 6的重新实现。在每个lecture中，我们将会针对xv6的其中一个部分，学习它的源代码；布置的homwork将会辅助你准备这些lectures。在这部分的结尾（大于到一半的样子），你将会理解一个基于Intel PC的良好设计的操作系统的源代码，这将会帮助你构建你自己的操作系统。


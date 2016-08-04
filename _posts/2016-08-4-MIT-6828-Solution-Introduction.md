---
layout: post
title: MIT 6.828课程解析（一） 简介
---

**本文翻译自[MIT 6.828 overview](https://pdos.csail.mit.edu/6.828/2014/overview.html)**

6.828讲授设计或实现操作系统的基础概念。你将会深入学习到以下概念：

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

为了掌握这些概念，6.828由三部分组成：讲义（lectures）、阅读（readings）以及一个大型试验。讲义和阅读是为了让你熟悉主要的概念。试验迫使你深层次的理解这些概念，因为你将从零开始会构建一个操作系统。在试验之后，你将领会到诸如“降低复杂度”和“概念完整性”等设计目标的意义。


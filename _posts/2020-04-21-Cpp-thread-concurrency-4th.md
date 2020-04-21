---
title: C++多线程并发系列（四）
subtitle: C++ Concurrency in Action 2nd Edition 读书笔记
date: 2020-04-21 14:18:15
categories: C++
tags: [C++, 并发控制]
---

# 概述

C++ 是一种系统编程语言，标准委员会的目标之一是，将不需要比 C++ 更底层的语言。C++ 应该为程序员提供足够的灵活性，使他们能够在没有语言妨碍的情况下执行所需的任何操作，从而在需要时允许他们“靠近机器”。`原子类型和操作`允许这样做，从而为低级同步操作提供了便利，这些操作通常会减少为一到两个CPU指令。

在本章中，我将首先介绍内存模型的基础知识，然后继续介绍原子类型和操作，最后介绍原子类型操作可使用的各种同步类型。这非常复杂：除非您打算编写使用原子操作进行同步的代码（例如无锁数据结构），否则您无需了解这些详细信息。

# 内存模型

内存模型有两个方面：基本结构方面（与事物在内存中的布局方式有关）和并发方面。结构方面对于并发非常重要，尤其是在您研究低级原子操作时，因此我将从这些方面开始。在 C++ 中，所有内容都与对象和内存位置有关。

## 对象与内存位置

C++ 程序中的所有数据均由对象组成，C++ 标准将对象定义为“存储区域”，尽管它继续为这些对象分配属性，例如它们的类型和生命周期。

这些对象中的一些是基本类型的简单值，例如int或float，而另一些是用户定义类的实例。有些对象（例如数组，派生类的实例以及具有非静态数据成员的类的实例）具有子对象，而其他对象则没有。无论其类型如何，一个对象都存储在一个或多个存储位置中。

* 每个变量都是一个对象，包括那些属于其他对象的成员。

* 每个对象至少占用一个内存位置。

* 基本类型的变量（例如int或char）无论它们大小如何，都恰好占据一个内存位置，即使它们是相邻的或数组的一部分。

* 相邻的 bit fields 是同一内存位置的一部分。

## 对象，内存位置与并发

C++ 中的多线程应用程序至关重要的部分：一切都取决于这些内存位置。如果有两个线程访问单独的内存位置，那就没问题：一切正常。另一方面，如果两个线程访问相同的内存位置，则必须小心。如果两个线程都没有更新内存位置，那很好。只读数据不需要保护或同步。如果任何一个线程正在修改数据，则存在 race condition（争用情况）的可能性。

为了避免争用情况，线程就要规定执行顺序。一种方式是使用 mutex，后一线程必须等待前一线程解锁。第二种方式是使用原子操作来避免竞争访问同一内存位置而导致的未定义行为。

每个对象从初始化开始都有一个修改顺序，这个顺序由来自所有线程对该对象的写操作组成。通常这个顺序在运行时会变动，但在任何给定的程序执行中，系统中所有线程都必须遵循此顺序。

如果对象不是原子类型，你有责任确保有足够的同步来确保线程同意每个变量的修改顺序。如果不同的线程看到单个变量的值的不同序列，则说明存在数据争用和未定义的行为。使用原子操作就可以把同步的责任抛给编译器。

# C++ 的原子类型与操作

## 标准原子类型

[标准原子类型](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/yuan-zi-cao-zuo-he-yuan-zi-lei-xing/biao-zhun-yuan-zi-lei-xing)

## std::atomic_flag

[std::atomic_flag](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/yuan-zi-cao-zuo-he-yuan-zi-lei-xing/std-atomic_flag)

## 其他原子类型

[其他原子类型](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/yuan-zi-cao-zuo-he-yuan-zi-lei-xing/qi-ta-yuan-zi-lei-xing)

# 同步操作和强制排序

## synchronizes-with relationship

[synchronizes-with](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/synchronizes-with)

## happens-before relationship

[happens-before](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/happens-before)

[inter-thread happens-before](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/inter-thread-happens-before)

[strongly-happens-before](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/strongly-happens-before)

## Memory ordering for atomic operations

[std::memory_order](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/std-memory_order)

### Relaxed ordering

[Relaxed ordering](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/std-memory_order/relaxed-ordering)

### Release-Consume ordering

[Release-Consume ordering](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/std-memory_order/release-consume-ordering)

### Release-Acquire ordering

[Release-Acquire ordering](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/std-memory_order/release-acquire-ordering)

### Sequentially-consistent ordering

[Sequentially-consistent ordering](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/std-memory_order/sequentially-consistent-ordering)

## Fences

[std::atomic_thread_fence](https://downdemo.gitbook.io/cpp-concurrency-in-action-2ed/4.-c++-nei-cun-mo-xing-he-ji-yu-yuan-zi-lei-xing-de-cao-zuo-the-c++-memory-model-and-operations-on-a/tong-bu-cao-zuo-he-qiang-zhi-pai-xu-enforced-ordering/std-atomic_thread_fence)
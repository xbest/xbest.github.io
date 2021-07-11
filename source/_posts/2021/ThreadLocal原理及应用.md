---
title: 浅析ThreadLocal原理、实现及适用场景
date: 2021/07/11
categories:
 - [Java]
 - [源码浅析]
urlName: ThreadLocal
---
本文主要从ThreadLocal的源代码出发，分析了以下几个问题：
- ThreadLocal的原理及实现， 
- ThreadLocal到底存不存在内存泄漏
- ThreadLocal在开源代码中的应用示例
- 如何正确的使用ThreadLocal

<!-- more -->

## ThreadLocal原理及实现
### ThreadLocal定义
要研究ThreadLocal，我们先要弄明白ThreadLocal是什么？先来看一下源代码中官方给出的描述：
> This class provides thread-local variables. 
> These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own,
> independently initialized copy of the variable. <br>
> ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).
> Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; 
> after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).

官方给出的定义是ThreadLocal提供了**线程本地的变量**。它与普通的变量的区别在于，每个使用该变量的线程都会初始化一份**完全独立的实例副本**。

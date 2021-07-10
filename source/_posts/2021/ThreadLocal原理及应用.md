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
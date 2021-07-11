---
title: 浅析ThreadLocal原理、实现及适用场景
date: 2021/07/11
categories:
 - [Java]
 - [源码浅析]
urlName: ThreadLocal
---
本文主要从`ThreadLocal`的源代码出发，分析了以下几个问题：
- `ThreadLocal`的定义、原理及实现
- `ThreadLocal`到底存不存在内存泄漏
- `ThreadLocal`在开源代码中的应用示例
- `ThreadLocal`的适用场景

<!-- more -->

## ThreadLocal定义、原理及实现
### ThreadLocal定义
要研究`ThreadLocal`，我们先要弄明白`ThreadLocal`是什么？先来看一下源代码中官方给出的描述：
> This class provides thread-local variables. 
> These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own,
> independently initialized copy of the variable. <br>
> ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).
> Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; 
> after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).

官方给出的定义是ThreadLocal提供了**线程本地的变量**。它与普通的变量的区别在于，每个使用该变量的线程都会初始化一份**完全独立的实例副本**。
[Java中总共定义了以下四种变量](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/variables.html)：
- Instance Variables(Non-Static Fields)
- Class Variables(Static Fields)
- Local Variables
- Parameters
  
那么什么是`thread-local variables`呢？从字面意思来看就是`Local Variables`，就是本地变量或者局部变量，但是本地变量的作用域是方法内部，也就是`method-local variables`，所以从这个角度来分析就是作用域不同，`thread-local variables`的作用域就是thread内部，也就是不局限于方法内部，所以`ThreadLocal`的作用域就是整个thread内。
  
### ThreadLocal原理
![ThreadLocal原理图](https://raw.githubusercontent.com/xbest/image-hosting/main/img/20210711155545.jpg)
如上图所示，`ThreadLocal`变量其实仅仅是作为`ThreadLocalMap`中的key来存储数据的，即`Thread`中的`ThreadLocalMap`类型的`threadLocals`字段才是真正存储数据的地方。
### ThreadLocal实现
首先看一下`ThreadLocal`的源代码，主要方法为`set`及`get`方法。
![ThreadLocal源码图](https://raw.githubusercontent.com/xbest/image-hosting/main/img/20210711142456.png)
如上图所示，`ThreadLocal`中并没有存储变量的字段，那么调用`ThreadLocal.set`方法，将变量存储到哪里了呢？
```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```
通过`set`方法源代码可以看到，真正存储变量数据信息的`ThreadLocalMap`不在`ThreadLocal`中，而是根据`Thread.currentThread()`查询或者创建的。
那么`getMap`和`createMap`是怎么实现的呢？我们继续跟踪查看这两方法的实现。
```java
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
`getMap`的方法简单到不能再简单了，就是直接返回了`Thread.currentThread().threadLocals`，非常清晰地能够看出真正存储变量数据的地方是当前线程的字段。
```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
`createMap`的方法也是简单到了极致，直接就是给当前线程的`threadLocals`字段赋值一个新创建的`ThreadLocalMap`对象的实例。
通过`set`和`get`两个关键方法的源代码可以基本上看出`ThreadLocal`的实现了，也真正明白了的意思了。
> 每个使用`ThreadLocal`的线程都独立初始化了一份实例的副本

## ThreadLocal到底存不存在内存泄漏
### 什么是内存泄漏（memory leak）
> A Memory Leak is a situation when there are objects present in the heap that are no longer used,
> but the garbage collector is unable to remove them from memory and, thus they are unnecessarily maintained.

![memory leak](https://raw.githubusercontent.com/xbest/image-hosting/main/img/20210711161412.webp)
内存泄漏就是heap中的对象已经没有其它地方在使用了，但是GC却不能回收这块内存，总结就是以下两点：
- 线程或者进程中没有地方**真正**在使用该对象
- GC不能回收该对象
结合上述两点内存泄漏的特征，我们针对`ThreadLocal`进行分析以下是否存在内存泄漏。
  
 


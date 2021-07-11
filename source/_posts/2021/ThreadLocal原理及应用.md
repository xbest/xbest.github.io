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
> independently initialized copy of the variable.
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
public class ThreadLocal<T> {
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
}
```
通过`set`方法源代码可以看到，真正存储变量数据信息的`ThreadLocalMap`不在`ThreadLocal`中，而是根据`Thread.currentThread()`查询或者创建的。
那么`getMap`和`createMap`是怎么实现的呢？我们继续跟踪查看这两方法的实现。
```java
public class ThreadLocal<T> {
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
}
```
`getMap`的方法简单到不能再简单了，就是直接返回了`Thread.currentThread().threadLocals`，非常清晰地能够看出真正存储变量数据的地方是当前线程的字段。
```java
public class ThreadLocal<T> {
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
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
结合上述两点内存泄漏的特征，我们针对`ThreadLocal`进行分析以下是否存在内存泄漏。我们先看一下`ThreadLocal`的典型应用  
```java
   public class UserContext {
       // Thread local variable containing user
       private static final ThreadLocal<User> userThreadLocal = new ThreadLocal<User>();
  
       // Returns the current user
       public static User get() {
           return userThreadLocal.get();
       }
       
       public static void set(User user) {
           userThreadLocal.set(user);
       }
   }
```
假设我们有一个线程池，线程池里的线程每次都会处理`User`相关数据，假设我们从线程池中取出线程A，在线程A的第一个方法入口时调用`UserContext.set`方法，保存`User`信息，
然后在线程A中调用`UserContext.get`方法去获取`User`信息，在线程A退出时，即不需要当前`User`信息了，此时满足了内存泄漏的第一个条件即这个`User`不再被使用。
但是从`userThreadLocal`的角度来看，其实下次还是会被别的线程（例如线程B）使用的，即使来的新请求又被线程A所处理，那么这次线程A存储的`User`也不是上次的`User`了。
在代码中可以看到`ThreadLocal`是`private static`的，也就是`Class Variables`，只有当类卸载的时候才会被回收，所以GC不会回收`userThreadLocal`，
同时由于线程A在线程池中所以线程A也不会被释放，那么线程A所持有的`threadLocals`也不会被回收，那么`threadLocals`中所存储的`User`对象当然也就不会被GC回收，
满足了内存泄漏的第二个条件，从这个角度来看的话，确实存在内存泄漏。
但是内存泄漏的第一个条件改为**线程或者内存中不能通过引用访问到该对象**，那么此时就不满足内存泄漏了，因为即使线程A返回到线程池后，下次再进来的话还是能访问到该`User`对象的，
如果该`User`对象没有被覆盖的话。


 


---
layout: post
title: "ThreadLocal是否会引发内存泄露的问题"
date: 2018-04-02 19:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

网上有观点认为使用ThreadLocal会造成内存泄露的问题，那到底事实是不是这样的呢？

<!--more-->


## ThreadLocal的介绍
**<font color=pink>ThreadLocal 不是用来解决共享对象的多线程访问问题的，而是为每个线程创建一个单独的变量副本，提供了保持对象的方法和避免参数传递的复杂性</font>**。

ThreadLocal早期的设计是：定义一个map，将当前thread作为key。

现在ThreadLocal的设计思路是：
* 每个Thread维护一个ThreadLocalMap映射表，这个映射表的key是ThreadLocal实例本身，value是真正需要存储的Object。
* 这样设计的优势:
  * 每个Map的Entry数量变小了(之前是Thread的数量，现在是ThreadLocal的数量)
  * 当Thread销毁之后对应的ThreadLocalMap也就随之销毁了，能减少内存使用量。
  * 速度快了很多，因为如果把所有线程要用的对象都放到一个静态map中的话 多线程并发访问需要进行同步。

<font color=red>**注意**：ThreadLocalMap是使用ThreadLocal的弱引用作为Key的！</font>于是，对象之间的关系：
<img width=800 src="/assets/blogImg/Java_Basic/ThreadLocal/1.png">

其中，key是一个`WeakReference`对象，有一个弱引用指向 ThreadLocal，而 value 是一个强引用，指向用户设置的对象。


## 关于ThreadLocal会引发内存泄露的问题

有的观点认为ThreadLocal会引发内存泄露，理由如下:
1. ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用来引用它，那么系统 GC 的时候，这个ThreadLocal势必会被回收。
2. 这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value了。
3. 如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value `，使得value 无法回收，造成内存泄漏。 


实际上，ThreadLocalMap的设计中已经考虑到这种情况，也加上了一些防护措施:
在ThreadLocal的 `get()`、`set()`、`remove()`的时候都会清除线程ThreadLocalMap里所有key为null的value。

有了预防措施也不能保证不会内存泄漏：
 * 使用线程池的时候，这个线程执行任务结束，ThreadLocal对象被回收了，线程放回线程池中不销毁，这个线程一直不被使用，导致内存泄漏。
 * 分配使用了ThreadLocal又不再调用get(),set(),remove()方法，那么这个期间就会发生内存泄漏。

<font color=red>ThreadLocal内存泄漏的根源是</font>：<font color=blue>由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除map中的value就会导致内存泄漏,而不是因为弱引用</font>。

JDK的建议：<font color=green>将ThreadLocal变量定义成private static的，这样的话ThreadLocal的生命周期就更长，由于一直存在ThreadLocal的强引用，所以ThreadLocal也就不会被回收，也就能保证任何时候都能根据ThreadLocal的弱引用访问到Entry的value值，然后remove它，防止内存泄露</font>。
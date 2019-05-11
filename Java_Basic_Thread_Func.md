---
layout: post
title: "Java线程的几个方法"
date: 2017-11-02 21:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

简单介绍Java线程的几个方法。

<!--more-->

### Thread的interrupt()方法

如果线程T的interrupt()方法被调用，那么需要分以下几种情况：

#### 阻塞于 Object.wait()或join()或sleep(long)
`它的中断标志被清除,并且它会收到一个InterruptedException。`


#### 阻塞于 java.nio.channels.InterruptibleChannel上的I/O操作
`这个channel会被关闭，这个线程的中断标志被设置，这个线程会收到java.nio.channels.ClosedByInterruptException。`


#### 阻塞于 java.nio.channels.Selector
`线程会从selection操作马上返回，就像是Selector的wakeup方法被调用。`


#### 没有以上情况时
`线程的中断标志被设置。`



### Thread的interrupted()方法

测试这个线程的中断标志是否被设置(是否被interrupted),如果有，这个标志会被清除。


### Thread的isInterrupted()方法
测试这个线程的中断标志是否被设置(是否被interrupted),并且中断标志不会被改变。



### Thread的join()方法

当前线程A执行thread.join()方法,线程A会等待thread线程终止以后才会从join()方法返回。

join()方法的实现:

```bash
public final synchronized void join()throws InterruptedException{
  while (isAlive()) {
    wait(0);
  }
}
```
当thread线程终止时,它会通过调用自身的notifyAll()方法,来通知所有等待在本线程对象的线程。
这样，线程A才会从join()方法中返回。
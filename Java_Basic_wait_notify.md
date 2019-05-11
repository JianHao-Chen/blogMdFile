---
layout: post
title: "Java的wait和notify"
date: 2017-11-02 21:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

简单介绍Java的wait和notify，内容来自Java JDK的文档。

<!--more-->

### Object的wait()和notify()方法

#### 这2个方法的几个要点：
* 当前线程执行对象A的wait方法使当前线程的状态由 `RUNNABLE->WAITING`,直到有某个线程执行对象A的`notify()`或`notifyAll()`方法。
* 线程想执行对象A的`wait()`方法,必须先获取对象A的monitor锁。否则,会抛出`IllegalMonitorStateException`异常。
* 线程会释放对象的monitor锁,进入等待,直到某个线程执行`notify()`操作。线程还是要等待直到它再次获取对象的monitor锁。
* 执行`wait()`方法的线程进入对象A的等待队列并释放已获取的monitor锁，注意，仅仅是释放对象A的monitor锁，而线程T获取到的其他对象的monitor锁，仍然保持。
* <font color=green>对象A的`wait()`方法使得当前线程(称作T),把自己放到对象A的等待队列中,并且释放已获取的monitor锁。线程T变成不可调度的(处于休眠),直到以下3种情况之一发生:</font>
  * 某个线程执行对象A的`notify()`方法，并且线程T刚好被选中来唤醒
  * 某个线程执行对象A的`notifyAll()`方法
  * 某个线程执行T的`interrupt()`方法

随后,线程T被移出对象A的等待队列,并且可以重新参加调度。

<font color=red>注意：</font>
**线程T还是要与其他的线程竞争对象A的monitor锁，一旦它获取到锁,它才从wait()方法返回**

#### 这个方法的通常用法
```bash
synchronized (lockObject) {
  while (condition==false)
    obj.wait(timeout);
}
```

使用 while(condition)循环是因为
* spurious wakeup(虚假唤醒)导致线程被唤醒,但是等待的条件还是不满足。
* 多个线程同时`wait()`,有可能其他线程更早醒来修改了状态。
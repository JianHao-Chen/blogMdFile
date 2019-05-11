---
layout: post
title: "Java线程的状态"
date: 2017-11-02 21:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

简单介绍Java线程的状态。

<!--more-->

## 线程的状态

在给定的一个时刻,线程只能处于其中的一个状态:
1. <font color=green>NEW</font>
  初始状态,线程被创建,但是还没有调用start()方法。
2. <font color=green>RUNNABLE</font>
  运行状态,Java线程将操作系统中的就绪和运行2种状态笼统称作"运行中"。
3. <font color=green>BLOCKED</font>
  阻塞状态,表示线程阻塞于锁。
4. <font color=green>WAITING</font>
  等待状态,表示线程进入等待状态，进入该状态表示当前线程需要等待其他线程,做出一些特定的动作(通知或中断)。
5. <font color=green>TIME_WAITING</font>
  超时等待状态,该状态不同于 WAITING,它是可以在指定的时间自行返回的。
6. <font color=green>TERMINATED</font>
  终止状态,表示线程已经执行完毕。


## 线程的状态变迁

1. <font color=blue>NEW -> RUNNABLE</font> ：  Thread.start()

2. <font color=blue>RUNNING -> READY</font> ： Thread.yield()
<font color=blue>READY -> RUNNING</font> ： 系统调度

3. <font color=blue>RUNNABLE -> WAITING</font> ： Object.wait()  || Object.join()
<font color=blue>WAITING -> RUNNABLE</font> ： Object.notify() || Object.notifyAll()

4. <font color=blue>RUNNABLE -> TIME_WAITING</font> ： Thread.sleep() || Object.wait(long)
<font color=blue>TIME_WAITING -> RUNNABLE</font> ： Object.notify() || Object.notifyAll()

5. <font color=blue>RUNNABLE -> BLOCKED</font> ： 等待进入synchronized块/方法
<font color=blue>BLOCKED -> RUNNABLE</font> ： 获取到锁 <br/><br/><br/>



## 用Java代码把线程的各种状态显示出来

```bash
public class ThreadState {

  public static void main(String[] args) {
  
    // Thread.State: TIMED_WAITING (sleeping)
    new Thread(new TimeWaiting(),"TimeWaitingThread").start();
    
    // Thread.State: WAITING (on object monitor)
    new Thread(new Waiting(),"WaitingThread").start();
    
    
    // 使用2个Blocked线程,一个获取锁成功,另一个被阻塞
    
    // Thread.State: TIMED_WAITING (sleeping)
    new Thread(new Blocked(),"BlockedThread-1").start();
    // Thread.State: BLOCKED (on object monitor)
    new Thread(new Blocked(),"BlockedThread-2").start();
    
  }
  
  // 该线程不断的进行睡眠
  static class TimeWaiting implements Runnable{
    public void run(){
      while(true){
        try{
          TimeUnit.SECONDS.sleep(100);
        }catch(InterruptedException e){}
      }
    }
  }
  
  // 该线程在Waiting.class实例上等待
  static class Waiting implements Runnable{
    public void run(){
      while(true){
        synchronized(Waiting.class){
          try{
            Waiting.class.wait();
          }
          catch(InterruptedException e){
            e.printStackTrace();
          }
        }
      }
    }
  }
  
  //该线程在Blocked.class实例上加锁后,不会释放该锁
  static class Blocked implements Runnable{
    public void run(){
      synchronized(Blocked.class){
        while(true){
          try{
            TimeUnit.SECONDS.sleep(100);
          }catch(InterruptedException e){}
        }
      }
    }
  }

}

```
---
layout: post
title: "Selector的wakeup方法"
date: 2018-08-06 17:36
comments: false
tags: 
- Java
- NIO
categories:	
- 基础
- Java
---


在Java的Selector接口中，定义了wakeup方法：

> Causes the first selection operation that has not yet returned to return immediately.
>
> If another thread is currently blocked in an invocation of the `select()` or `select(long)` methods then that invocation will return immediately. If no selection operation is currently in progress then the next invocation of one of these methods will return immediately unless the `selectNow()` method is invoked in the meantime. In any case the value returned by that invocation may be non-zero. Subsequent invocations of the `select()` or `select(long)` methods will block as usual unless this method is invoked again in the meantime.
>
> Invoking this method more than once between two successive selection operations has the same effect as invoking it just once.

🌟 简单来说，就是如果一个线程调用select方法是会阻塞的，如果其他线程想终止这个过程，可以调用wakeup方法将这个线程“唤醒”。

<!--more-->

### 🔥 wakeup方法的实现

#### Linux下的实现

linux下Selector默认实现为PollSelectorImpl，当内核版本大于2.6时，实现为EPollSelectorImpl。

##### PollSelector的相关细节
代码如下：
```
class PollSelectorImpl
    extends AbstractPollSelectorImpl
{

    // File descriptors used for interrupt
    private int fd0;
    private int fd1;

    // Lock for interrupt triggering and clearing
    private Object interruptLock = new Object();
    private boolean interruptTriggered = false;

    /**
     * Package private constructor called by factory method in
     * the abstract superclass Selector.
     */
    PollSelectorImpl(SelectorProvider sp) {
        super(sp, 1, 1);
        long pipeFds = IOUtil.makePipe(false);
        fd0 = (int) (pipeFds >>> 32);
        fd1 = (int) pipeFds;
        pollWrapper = new PollArrayWrapper(INIT_CAP);
        pollWrapper.initInterrupt(fd0, fd1);
        channelArray = new SelectionKeyImpl[INIT_CAP];
    }
    
    
    // 省略部分代码
    
    
    public Selector wakeup() {
        synchronized (interruptLock) {
            if (!interruptTriggered) {
                pollWrapper.interrupt();
                interruptTriggered = true;
            }
        }
        return this;
    }
}
```
细节：
(1) 其中interruptTriggered为中断已触发标志，当pollWrapper.interrupt()之后，该标志即为true了；得益于这个标志，连续两次wakeup，只会有一次效果。
(2) 通过使用IOUtil的makePipe方法创建管道，fd0为read端，fd1为write端。
(3) PollArrayWrapper中的initInterrupt()方法初始化fd0、fd1，代码如下：
```
//putDescriptor(0, fd0)初始化pollfd数组中的第一个pollfd,即指PollSelector关注的第一个fd，即为fd0；
//putEventOps(0, POLLIN)初始化fd0对应pollfd中的events为POLLIN,即指fd0对可读事件感兴趣；
//putReventOps(0, 0)只是初始化一下fd0对应的pollfd中的revents;
void initInterrupt(int fd0, int fd1) {
    interruptFD = fd1;
    putDescriptor(0, fd0);
    putEventOps(0, POLLIN);
    putReventOps(0, 0);
}
```
(4) PollArrayWrapper中的interrupt()方法如下：
```
public void interrupt() {
    interrupt(interruptFD);
}
```
这个interupt是native方法，找到它的实现（在PollArrayWrapper.c）：
```
JNIEXPORT void JNICALL
Java_sun_nio_ch_PollArrayWrapper_interrupt(JNIEnv *env, jobject this, jint fd)
{
    int fakebuf[1];
    fakebuf[0] = 1;
    if (write(fd, fakebuf, 1) < 0) {
         JNU_ThrowIOExceptionWithLastError(env,
                                          "Write to interrupt fd failed");
    }
}
```
可以看到，这里是调用write方法，向fd写入1个字节。



##### EPollSelector的相关细节
EPollSelector与PollSelector相比，只是initInterrupt方法的不同：
```
 void initInterrupt(int fd0, int fd1) {
    outgoingInterruptFD = fd1;
    incomingInterruptFD = fd0;
    epollCtl(epfd, EPOLL_CTL_ADD, fd0, EPOLLIN);
}
```
它是通过native方法epollCtl，实现对fd0添加读感兴趣事件的。



#### Windows下的实现
window下Selector的实现为WindowsSelectorImpl，代码如下：
```
final class WindowsSelectorImpl extends SelectorImpl {

    // 。。。省略部分代码

    WindowsSelectorImpl(SelectorProvider sp) throws IOException {
        super(sp);
        pollWrapper = new PollArrayWrapper(INIT_CAP);
        wakeupPipe = Pipe.open();
        wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();
    
        // Disable the Nagle algorithm so that the wakeup is more immediate
        SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
        (sink.sc).socket().setTcpNoDelay(true);
        wakeupSinkFd = ((SelChImpl)sink).getFDVal();
    
        pollWrapper.addWakeupSocket(wakeupSourceFd, 0);
    }
    
    
     // 。。。省略部分代码

    public Selector wakeup() {
        synchronized (interruptLock) {
            if (!interruptTriggered) {
                setWakeupSocket();
                interruptTriggered = true;
            }
        }
        return this;
    }

}
```
windows下PollArrayWrapper的相关方法：
```
// Adds Windows wakeup socket at a given index.
void addWakeupSocket(int fdVal, int index) {
    putDescriptor(index, fdVal);
    putEventOps(index, POLLIN);
}

// Sets Windows wakeup socket to a signaled state.
private void setWakeupSocket() {
    setWakeupSocket0(wakeupSinkFd);
}
```

细节：
（1）虽然看到了`Pipe.open`，但是这个实际上是通过socket实现的，并不像linux下的管道。
（2）addWakeupSocket方法对wakeupSourceFd添加了读事件感兴趣。
（3）setWakeupSocket方法调用的是native方法，对应的方法在WindowsSelectorImpl.c文件下：
```
JNIEXPORT void JNICALL
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,
                                                jint scoutFd)
{
    /* Write one byte into the pipe */
    const char byte = 1;
    send(scoutFd, &byte, 1, 0);
}
```
可以看到，也是对socket写入1个字节。



### ⚠️ 总结
1. Linux下，通过为PollSelector实例创建管道，将其中的读端添加读事件感兴趣。当需要唤醒selector时，往写端写入1个字节，就达到目的了。
2. Windows下，通过创建一对socket，其他与Linux下的差不多。


####  🍎 补充

##### Java的Selector的wakeup为什么要这样设计？
在linux/c程序中，一个阻塞在select上的线程有以下三种方式可以被唤醒：
1. 有数据可读/写，或出现异常。
2. 阻塞时间到，即time out。 
3. 收到一个non-block的信号。

第2点，等待阻塞时间到，这算什么“唤醒”，排除！
第3点，信号机制，不具备跨平台特性（windows没有信号机制？这个待查证）
所以，只能用第1种方法了。

##### Mina中的1个机制
1. Mina框架会创建一个Work对象的线程。
2. Work对象的线程的run()方法会从一个队列中拿出一堆Channel，然后使用Selector.select()方法来侦听是否有数据可以读/写。
3. 最关键的是，在select的时候，如果队列有新的Channel加入，那么，Selector.select()会被唤醒，然后重新select最新的Channel集合。
4. 要唤醒select方法，只需要调用Selector的wakeup()方法。

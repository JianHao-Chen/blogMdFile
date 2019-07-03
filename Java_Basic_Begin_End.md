---
layout: post
title: "Selector和Channel是怎么知道线程被中断的？"
date: 2018-08-10 14:36
comments: false
tags: 
- Java
- NIO
categories:	
- 基础
- Java
---


Java的Channel和Selector均有自己的`begin()`和`end()`方法。
<font color=pink>**当一个线程阻塞于Selector或Channel的I/O操作，如果此时这个线程被中断了，那么`begin()`和`end()`方法就是用于通知这个Selector或Channel**</font>。

<!--more-->

一般begin()  &  end()的用法如下：
```
try {
    begin();
    // Perform blocking I/O operation here
    ...
} finally {
    end();
}
```



#### 🌲 selector的 begin()  &  end()
`AbstractSelector`的begin()和end()方法：
```
/**
 * Marks the beginning of an I/O operation that might block indefinitely.
 *
 * <p> This method should be invoked in tandem with the {@link #end end}
 * method, using a <tt>try</tt>&nbsp;...&nbsp;<tt>finally</tt> block as
 * shown <a href="#be">above</a>, in order to implement interruption for
 * this selector.
 *
 * <p> Invoking this method arranges for the selector's {@link
 * Selector#wakeup wakeup} method to be invoked if a thread's {@link
 * Thread#interrupt interrupt} method is invoked while the thread is
 * blocked in an I/O operation upon the selector.  </p>
 */
protected final void begin() {
    if (interruptor == null) {
        interruptor = new Interruptible() {
                public void interrupt(Thread ignore) {
                    AbstractSelector.this.wakeup();
                }};
    }
    AbstractInterruptibleChannel.blockedOn(interruptor);
    Thread me = Thread.currentThread();
    if (me.isInterrupted())
        interruptor.interrupt(me);
}



/**
 * Marks the end of an I/O operation that might block indefinitely.
 *
 * <p> This method should be invoked in tandem with the {@link #begin begin}
 * method, using a <tt>try</tt>&nbsp;...&nbsp;<tt>finally</tt> block as
 * shown <a href="#be">above</a>, in order to implement interruption for
 * this selector.  </p>
 */
protected final void end() {
    AbstractInterruptibleChannel.blockedOn(null);
}
```

##### AbstractInterruptibleChannel的blockedOn方法
```
static void blockedOn(Interruptible intr) {         // package-private
    sun.misc.SharedSecrets.getJavaLangAccess().blockedOn(Thread.currentThread(),intr);
}
```
这里getJavaLangAccess就不研究了，简单的说就是，调用了 Thread 类的`blockedOn`方法：
```
/* The object in which this thread is blocked in an interruptible I/O
 * operation, if any.  The blocker's interrupt method should be invoked
 * after setting this thread's interrupt status.
 */
private volatile Interruptible blocker;
private final Object blockerLock = new Object();

/* Set the blocker field; invoked via sun.misc.SharedSecrets from java.nio code
 */
void blockedOn(Interruptible b) {
    synchronized (blockerLock) {
        blocker = b;
    }
}
```
原来Thread类中包含Interruptible的私有成员blocker，blockedOn只是给它赋值而已。

继续看 Thread 的`interrupt`方法：
```
/**
 * Interrupts this thread.
 *
 * ..... 省略
 *
 * <p> If this thread is blocked in an I/O operation upon an {@link
 * java.nio.channels.InterruptibleChannel InterruptibleChannel}
 * then the channel will be closed, the thread's interrupt
 * status will be set, and the thread will receive a {@link
 * java.nio.channels.ClosedByInterruptException}.
 *
 * <p> If this thread is blocked in a {@link java.nio.channels.Selector}
 * then the thread's interrupt status will be set and it will return
 * immediately from the selection operation, possibly with a non-zero
 * value, just as if the selector's {@link
 * java.nio.channels.Selector#wakeup wakeup} method were invoked.
 * ..... 省略
 */     
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}
```
关键点在`b.interrupt(this);`这行代码，它调用了`Interruptible`的`interrupt`方法，在`AbstractSelector`里面就是`AbstractSelector.this.wakeup();`。这个跟注释里说的一样，相当于调用了 Selector的wakeUp方法。

##### 小结
begin和end方法都用到了`AbstractInterruptibleChannel.blockOn()`：我们可以理解为，如果线程被中断，就附带执行我的这个interrupt方法吧。




#### 🌲 Channel的 begin()  &  end()

AbstractInterruptibleChannel的 begin()  &  end()：
```
protected final void begin() {
    if (interruptor == null) {
        interruptor = new Interruptible() {
                public void interrupt(Thread target) {
                    synchronized (closeLock) {
                        if (!open)
                            return;
                        open = false;
                        interrupted = target;
                        try {
                            AbstractInterruptibleChannel.this.implCloseChannel();
                        } catch (IOException x) { }
                    }
                }};
    }
    blockedOn(interruptor);
    Thread me = Thread.currentThread();
    if (me.isInterrupted())
        interruptor.interrupt(me);
}

protected final void end(boolean completed)
        throws AsynchronousCloseException
{
    blockedOn(null);
    Thread interrupted = this.interrupted;
    if (interrupted != null && interrupted == Thread.currentThread()) {
        interrupted = null;
        throw new ClosedByInterruptException();
    }
    if (!completed && !open)
        throw new AsynchronousCloseException();
}
```
这里，AbstractInterruptibleChannel是通过implCloseChannel()方法来实现Channel的关闭的。



#### 🌲 总结
1. begin()、end()都用到了`AbstractInterruptibleChannel.blockedOn(Interruptible intr)`方法，begin方法其实是赋值Thread类的私有变量 blocker，而end方法是将bloacker置为null。
2. 当Thread被中断了，会调用这个blocker的interrupt()方法，这个方法调用`Selector.wakeup()`或`AbstractInterruptibleChannel.implCloseChannel()`，来实现通知Selector、关闭通道。
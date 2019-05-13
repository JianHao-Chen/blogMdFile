---
layout: post
title: "Java的Channel是如何关闭的"
date: 2018-08-07 14:36
comments: false
tags: 
- Java
- NIO
categories:	
- 基础
- Java
---

这篇文章介绍Java的Channel是如何关闭的。


<!--more-->

 📖 Java的`InterruptibleChannel`接口的定义：
> A channel that can be asynchronously closed and interrupted.
> 
> A channel that implements this interface is asynchronously closeable : If a thread is blocked in an I/O operation on an interruptible channel then another thread may invoke the channel's `close` method. This will cause the blocked thread to receive an `AsynchronousCloseException`.
>
> A channel that implements this interface is also interruptible : If a thread is blocked in an I/O operation on an interruptible channel then another thread may invoke the blocked thread's `interrupt` method. This will cause the channel to be closed, the blocked thread to receive a `ClosedByInterruptException` , and the blocked thread's interrupt status to be set.
>
> If a thread's interrupt status is already set and it invokes a blocking If a thread's interrupt status is already set and it invokes a blocking will immediately receive a `ClosedByInterruptException`; its interrupt status will remain set.
>
> A channel supports asynchronous closing and interruption if, and only if, it implements this interface.  This can be tested at runtime, if necessary, via the instanceof operator.


### 🍕 close方法

#### AbstractInterruptibleChannel的close方法

```
public final void close() throws IOException {
    synchronized (closeLock) {
        if (!open)
            return;
        open = false;
        implCloseChannel();
    }
}
```
相关逻辑都在`implCloseChannel`方法里面。

#### AbstractSelectableChannel的implCloseChannel方法
```
 /**
  * Closes this channel.
  *
  * <p> This method, which is specified in the {@link
  * AbstractInterruptibleChannel} class and is invoked by the {@link
  * java.nio.channels.Channel#close close} method, in turn invokes the
  * {@link #implCloseSelectableChannel implCloseSelectableChannel} method in
  * order to perform the actual work of closing this channel.  It then
  * cancels all of this channel's keys.  </p>
  */
protected final void implCloseChannel() throws IOException {
    implCloseSelectableChannel();
    synchronized (keyLock) {
        int count = (keys == null) ? 0 : keys.length;
        for (int i = 0; i < count; i++) {
            SelectionKey k = keys[i];
            if (k != null)
                k.cancel();
        }
    }
}
```
💦 这个方法做了2个事情：
1. 调用`implCloseSelectableChannel`（抽象方法），真正的做关闭通道的事情。
2. 调用注册在这个通道上的SelectionKey的`cancel`方法。

#### ServerSocketChannelImpl的implCloseSelectableChannel方法
```
protected void implCloseSelectableChannel() throws IOException {
    synchronized (stateLock) {
        nd.preClose(fd);
        long th = thread;
        if (th != 0)
            NativeThread.signal(th);
        if (!isRegistered())
            kill();
    }
}
```

#### SocketChannelImpl的implCloseSelectableChannel方法
```
// AbstractInterruptibleChannel synchronizes invocations of this method
// using AbstractInterruptibleChannel.closeLock, and also ensures that this
// method is only ever invoked once.  Before we get to this method, isOpen
// (which is volatile) will have been set to false.
//
protected void implCloseSelectableChannel() throws IOException {
    synchronized (stateLock) {
        isInputOpen = false;
        isOutputOpen = false;

        // Close the underlying file descriptor and dup it to a known fd
        // that's already closed.  This prevents other operations on this
        // channel from using the old fd, which might be recycled in the
        // meantime and allocated to an entirely different channel.
        //
        nd.preClose(fd);

        // Signal native threads, if needed.  If a target thread is not
        // currently blocked in an I/O operation then no harm is done since
        // the signal handler doesn't actually do anything.
        //
        if (readerThread != 0)
            NativeThread.signal(readerThread);

        if (writerThread != 0)
            NativeThread.signal(writerThread);

        // If this channel is not registered then it's safe to close the fd
        // immediately since we know at this point that no thread is
        // blocked in an I/O operation upon the channel and, since the
        // channel is marked closed, no thread will start another such
        // operation.  If this channel is registered then we don't close
        // the fd since it might be in use by a selector.  In that case
        // closing this channel caused its keys to be cancelled, so the
        // last selector to deregister a key for this channel will invoke
        // kill() to close the fd.
        //
        if (!isRegistered())
            kill();
    }
}
```
想要明白`ServerSocketChannelImpl`和`SocketChannelImpl`的implCloseSelectableChannel方法，需要搞明白<font color=orange>NativeDispatcher</font>和<font color=orange>NativeThread</font>。


### 🍔 NativeThread
NativeThread比较简单，注释也多：
```
// Signalling operations on native threads
//
// On some operating systems (e.g., Linux), closing a channel while another
// thread is blocked in an I/O operation upon that channel does not cause that
// thread to be released.  This class provides access to the native threads
// upon which Java threads are built, and defines a simple signal mechanism
// that can be used to release a native thread from a blocking I/O operation.
// On systems that do not require this type of signalling, the current() method
// always returns -1 and the signal(long) method has no effect.


class NativeThread {

    // Returns an opaque token representing the native thread underlying the
    // invoking Java thread.  On systems that do not require signalling, this
    // method always returns -1.
    //
    static native long current();

    // Signals the given native thread so as to release it from a blocking I/O
    // operation.  On systems that do not require signalling, this method has
    // no effect.
    //
    static native void signal(long nt);

    static native void init();

    static {
        Util.load();
        init();
    }

}
```

#### NativeThread的本地方法实现
```
// 自定义中断信号，kill –l
#define INTERRUPT_SIGNAL (__SIGRTMAX - 2)

//自定义的信号处理函数，当前函数什么都不做
static void
nullHandler(int sig)
{
}


/** --------- init方法的本地实现  ---------*/
JNIEXPORT void JNICALL
Java_sun_nio_ch_NativeThread_init(JNIEnv *env, jclass cl)
{
#ifdef __linux__

    /* Install the null handler for INTERRUPT_SIGNAL.  This might overwrite the
     * handler previously installed by java/net/linux_close.c, but that's okay
     * since neither handler actually does anything.  We install our own
     * handler here simply out of paranoia; ultimately the two mechanisms
     * should somehow be unified, perhaps within the VM.
     */

    sigset_t ss;
    struct sigaction sa, osa;
    sa.sa_handler = nullHandler;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    if (sigaction(INTERRUPT_SIGNAL, &sa, &osa) < 0)
        JNU_ThrowIOExceptionWithLastError(env, "sigaction");

#endif
}


/**  --------- current方法的本地实现  --------- */
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_NativeThread_current(JNIEnv *env, jclass cl)
{
#ifdef __linux__
    return (long)pthread_self();
#else
    return -1;
#endif
}



/**  --------- signal方法的本地实现  --------- */
JNIEXPORT void JNICALL
Java_sun_nio_ch_NativeThread_signal(JNIEnv *env, jclass cl, jlong thread)
{
#ifdef __linux__
    if (pthread_kill((pthread_t)thread, INTERRUPT_SIGNAL))
        JNU_ThrowIOExceptionWithLastError(env, "Thread signal failed");
#endif

```
💧 小结：
1. `init`方法用到了sigaction安装一个信号，就是当收到信号 INTERRUPT_SIGNAL 时，执行函数 nullHandler（空函数）。
2. `current`方法使用了`pthread_self()`获取当前线程ID。
3. `signal`方法使用了`pthread_kill`发送信号 INTERRUPT_SIGNAL
4. 其中的原理是：如果一个线程阻塞，其他线程向他发送信号，从而终止阻塞。



#### NativeDispatcher
在之前列出的`ServerSocketChannelImpl`的代码中，nd就是`NativeDispatcher`。
重新列出之前`implCloseSelectableChannel`的代码：
```
protected void implCloseSelectableChannel() throws IOException {
    synchronized (stateLock) {
        nd.preClose(fd);
        long th = thread;
        if (th != 0)
            NativeThread.signal(th);
        if (!isRegistered())
            kill();
    }
}
```
接下来介绍NativeDispatcher的preClose()和kill()方法。

#### 为什么需要preClose？
在多线程环境下，总是很难知道什么时候可安全的关闭或释放资源(如fd)，当一个线程A使用fd来读写，而另一个线程B关 闭或释放了fd，则A线程就会读写一个错误的文件或socket；为了防止这种情况出现，于是NIO就采用了经典的two-step处理方案：
1. 创建一个socket pair，假设FDs为sp[2]，先close掉sp[1]，这样，该socket pair就成为了一个半关闭的链接；复制(dup2)sp[0]到fd(即为我们想关闭或释放的fd)，这个时候，其他线程如果正在读写立即会获得EOF 或者Pipe Error，read或write方法里会检测这些状态做相应处理
2. 最后一个会使用到fd的线程负责释放


#### NativeDispatcher的init
```
/* File descriptor to which we dup other fd's before closing them for real */
static int preCloseFD = -1;


JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_init(JNIEnv *env, jclass cl)
{
    int sp[2];
    if (socketpair(PF_UNIX, SOCK_STREAM, 0, sp) < 0) {
        JNU_ThrowIOExceptionWithLastError(env, "socketpair failed");
        return;
    }
    preCloseFD = sp[0];
    close(sp[1]);
}
```
这个方法通过使用`socketpair`创建一个全双工链接，然后关闭sp[1]端。

#### NativeDispatcher的preclose
```
JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_preClose0(JNIEnv *env, jclass clazz, jobject fdo)
{
    jint fd = fdval(env, fdo);
    if (preCloseFD >= 0) {
        if (dup2(preCloseFD, fd) < 0)
            JNU_ThrowIOExceptionWithLastError(env, "dup2 failed");
    }
}
```
这个方法通过使用`dup2`将preCloseFD复制到fd(预关闭的fd，既fd被替换了，此时，其他线程对fd的读写，实际上是对preCloseFD进行读写，是会收到EOF或 Pipe Error)。

#### NativeDispatcher的close
这个才是真正关闭fd的操作。

在`implCloseSelectableChannel`里面，通过判断如果没有selector注册到当前通道，那么就可以调用`kill()`方法了：
```
public void kill() throws IOException {
    synchronized (stateLock) {
        if (state == ST_KILLED)
            return;
        if (state == ST_UNINITIALIZED) {
            state = ST_KILLED;
            return;
        }
        assert !isOpen() && !isRegistered();
        nd.close(fd);
        state = ST_KILLED;
    }
}
```
确实是调用了NativeDispatcher的close()方法，然后自然是系统调用close(fd)。


### 🍟 总结
1. `AbstractInterruptibleChannel`的close方法使原来阻塞在此通道的线程不再阻塞，并且收到异常。
2. 通过使用`NativeThread`（它使用了系统的信号），来实现“唤醒”阻塞的线程。
3. 为了避免多个线程下共享的fd的关闭释放而导致某些线程可能读写“旧”fd的问题，java nio的实现采用了two-step的关闭，具体实现在`NativeDispatcher`的preclose和close方法。
---
layout: post
title: "NIO的异步Channel"
date: 2019-05-12 08:55
comments: false
tags: 
- NIO
- Java
categories:	
- NIO
---

对于异步IO操作的检测和控制，AsynchronousChannel提供2种方式：
* 通过返回一个 java.util.concurrent.Future 对象来实现
* 通过传递一个 java.nio.channels.CompletionHandler，来完成，它会定义在操作完毕后所执行的处理程序方法。

下面主要分析`AsynchronousSocketChannel`的read操作，研究为了实现“异步”，JDK帮我们做了什么。

<!--more-->

### 异步Read的API

#### API的介绍

##### 2个Read函数的定义
`AsynchronousSocketChannel`的read有如下2个read函数：
```
public abstract <A> void read(ByteBuffer dst,
                                  long timeout,
                                  TimeUnit unit,
                                  A attachment,
                                  CompletionHandler<Integer,? super A> handler);
								  
								  
public abstract Future<Integer> read(ByteBuffer dst);								  
```
如上面代码所示，异步Read操作的API可以分为2种：
* CompletionHandler：通过回调函数
* Future：通过返回代表异步操作结果的Future对象

其实，不止Read操作，其他的异步IO操作，都有这2种方式的API。

##### CompletionHandler的定义
```
/**
 * A handler for consuming the result of an asynchronous I/O operation.
 *
 * @param   <V>     The result type of the I/O operation
 * @param   <A>     The type of the object attached to the I/O operation
 *
 * @since 1.7
 */
public interface CompletionHandler<V,A> {

   /**
    * Invoked when an operation has completed.
    *
    * @param   result
    *          The result of the I/O operation.
    * @param   attachment
    *          The object attached to the I/O operation when it was initiated.
    */
    void completed(V result, A attachment);
    
   /**
    * Invoked when an operation fails.
    *
    * @param   exc
    *          The exception to indicate why the I/O operation failed
    * @param   attachment
    *          The object attached to the I/O operation when it was initiated.
    */
    void failed(Throwable exc, A attachment);
}
```

#### API的使用例子

##### CompletionHandler方式
```
public class AsynClient {
    AsynchronousSocketChannel asynSocketChannel	= null;
    ByteBuffer recvBuffer = ByteBuffer.allocate(100);

    /**
    * （1）创建 AsynchronousSocketChannel
    */
    public AsynClient(){
        try {
            asynSocketChannel = AsynchronousSocketChannel.open();
        } catch (Exception e){}
    }
	
    /**
    * （2） 创建TCP连接
    */	 
    public void connect(String host, int port){
        InetSocketAddress socketAddress = new InetSocketAddress(host, port);
        asynSocketChannel.connect(socketAddress, this, mConnectHandler);
    }
    //  connect 的回调函数
    private final CompletionHandler<Void, AsynClient> mConnectHandler =
            new CompletionHandler<Void, AsynClient>() {
                @Override
                public void completed(Void result, AsynClient attachment) {
                    attachment.read();
                }
                @Override
                public void failed(Throwable exc, AsynClient attachment) {
                    // log ....
                }
            };
	
	
    /**
    * （3）从通道读取数据
    */	
    public void read(){
        asynSocketChannel.read(recvBuffer, recvBuffer, mReadHandler);
    }
    //  read 的回调函数
    private final CompletionHandler<Integer, ByteBuffer> mReadHandler =
            new CompletionHandler<Integer, ByteBuffer>() {
                @Override
                public void completed(Integer result, ByteBuffer attachment) {

                    if (result < 0) {
                        System.out.println("Read EOF");
                    } else {
                        attachment.flip();
                        // 读取处理数据，省略
                    }
                }

                @Override
                public void failed(Throwable exc, ByteBuffer attachment) {
                    // log ...
                }
            };
}
```


##### Future方式
```
InetSocketAddress socketAddress = new InetSocketAddress(host, port);
Future connectFuture = asynSocketChannel.connect(socketAddress);

while (!connectFuture.isDone()) {
	// sleep
}

Future<Integer> readFuture = asynSocketChannel.read(recvBuffer);
while (!readFuture.isDone()) {
    // sleep
}
try {
    Integer num = readFuture.get();
    System.out.println("read byte num is "+ num);
}
catch (Exception e){}
```

#### 小结
比较上面2种异步Read API的使用：
* CompletionHandler方式：需要自定义回调函数，当Channel有数据，这个回调函数被调用（<font color=red>被那个线程调用？这个很重要，后面会说</font>），从而执行读取数据的逻辑
* Future方式：得到Future对象，通过Future对象知道Read操作的结果（是否有数据可读），然后执行读取数据的逻辑

### 异步Read的实现

#### AsynchronousSocketChannel 的创建

我们可以使用静态方法 open 创建一个 AsynchronousSocketChannel 对象，它的定义如下：
```
public static AsynchronousSocketChannel open() throws IOException{
	return open(null);
}
```
这是我们前面例子调用的方法，它又调用了如下重载的`open`方法：
```
public static AsynchronousSocketChannel open(AsynchronousChannelGroup group) throws IOException{
	AsynchronousChannelProvider provider = (group == null) ?
            AsynchronousChannelProvider.provider() : group.provider();
	return provider.openAsynchronousSocketChannel(group);
}
```
上面方法的逻辑是：
* 如果入参的 group 为 null，那么使用默认的（系统级别的）`AsynchronousChannelProvider`，否则使用入参的 group。
* 调用 provider 的 `openAsynchronousSocketChannel`方法

在Linux下，我们看到`LinuxAsynchronousChannelProvider::openAsynchronousSocketChannel`方法：
```
@Override
public AsynchronousSocketChannel openAsynchronousSocketChannel(AsynchronousChannelGroup group)
	throws IOException
{
	return new UnixAsynchronousSocketChannelImpl(toPort(group));
}
```
继续看`UnixAsynchronousSocketChannelImpl`的构造方法：
```
UnixAsynchronousSocketChannelImpl(Port port)
    throws IOException
{
    super(port);

    // set non-blocking
    try {
        IOUtil.configureBlocking(fd, false);
    } catch (IOException x) {
        nd.close(fd);
        throw x;
    }

    this.port = port;
    this.fdVal = IOUtil.fdVal(fd);

    // add mapping from file descriptor to this channel
    port.register(fdVal, this);
}
```
其中，Port是`AsynchronousChannelGroup`的子类（它同时是一个抽象类），与“IO多路复用”相关，它的默认实现是`EPollPort`，省略部分代码：
```
final class EPollPort extends Port{

	// epoll file descriptor
    private final int epfd;
	
	// socket pair used for wakeup
    private final int sp[];
	
	// address of the poll array passed to epoll_wait
    private final long address;
	
	EPollPort(AsynchronousChannelProvider provider, ThreadPool pool)
        throws IOException
    {
        super(provider, pool);

        // open epoll
        this.epfd = epollCreate();

        // create socket pair for wakeup mechanism
        int[] sv = new int[2];
        try {
            socketpair(sv);
            // register one end with epoll
            epollCtl(epfd, EPOLL_CTL_ADD, sv[0], POLLIN);
        } catch (IOException x) {
            close0(epfd);
            throw x;
        }
        this.sp = sv;

        // allocate the poll array
        this.address = allocatePollArray(MAX_EPOLL_EVENTS);
    }
}
```
使用`open`创建一个`AsynchronousSocketChannel`对象的逻辑可以总结为：
* 使用`open`方法时，用户可以自定义线程池，也可以使用默认的（系统级别的）
* 异步SocketChannel的实现中，有2个重要的属性：
  * port：默认实现是`EPollPort`
  * fd：Socket对应的fd
* `EPollPort`在构造函数中调用`epollCreate`，创建epollFd




#### AsynchronousSocketChannel的Read的实现

看到**Read**的实现（在 UnixAsynchronousSocketChannelImpl.java）：
```
/**
 * Initiates a read or scattering read operation
 */
@Override
<V extends Number,A> Future<V> implRead(boolean isScatteringRead,
                                            ByteBuffer dst,
                                            ByteBuffer[] dsts,
                                            long timeout,
                                            TimeUnit unit,
                                            A attachment,
                                            CompletionHandler<V,? super A> handler)
{

    // ... 省略部分代码

    PendingFuture<V,A> result = null;
    synchronized (updateLock) {
        this.isScatteringRead = isScatteringRead;
        this.readBuffer = dst;
        this.readBuffers = dsts;
        if (handler == null) {
            this.readHandler = null;
            result = new PendingFuture<V,A>(this, OpType.READ);
            this.readFuture = (PendingFuture<Number,Object>)result;
            this.readAttachment = null;
        } else {
            this.readHandler = (CompletionHandler<Number,Object>)handler;
            this.readAttachment = attachment;
            this.readFuture = null;
        }
        if (timeout > 0L) {
            this.readTimer = port.schedule(readTimeoutTask, timeout, unit);
        }
        this.readPending = true;
        updateEvents();
    }
	return result;
	
	// ... 省略部分代码
}
```
上面方法的逻辑是：
* 调用`updateEvents()`方法，通过调用`epollCtl()`方法，将`POLLIN` Event添加到当前 fd 。
* 根据传入的参数 handler是否为 null：
  * 不为null：即使用`CompletionHandler`方式
  * 为null：即使用`Future`方式，返回`Future`对象
  
<font color=red>**补充：**</font>
不论使用`CompletionHandler`方式，还是`Future`方式的Read函数，最终都是调用到这个`implRead()`方法，不同之处在处理这个方法返回的`Future`对象：
* `CompletionHandler`方式：忽略返回的`Future`对象
* `Future`方式：返回`Future`对象给用户


#### 获取就绪事件
前面说过，`EPollPort`与“IO多路复用”相关，现在看看它这部分的实现：
```
final class EPollPort
    extends Port
{

	// epoll file descriptor
    private final int epfd;
	
	private final ArrayBlockingQueue<Event> queue;

	EPollPort start() {
		startThreads(new EventHandlerTask());
		return this;
	}
	
	/*
     * Task to process events from epoll and dispatch to the channel's
     * onEvent handler.
     *
     * Events are retreived from epoll in batch and offered to a BlockingQueue
     * where they are consumed by handler threads. A special "NEED_TO_POLL"
     * event is used to signal one consumer to re-poll when all events have
     * been consumed.
     */
	private class EventHandlerTask implements Runnable {
        private Event poll() throws IOException {
            try {
                for (;;) {
                    int n = epollWait(epfd, address, MAX_EPOLL_EVENTS);
                    /*
                     * 'n' events have been read. Here we map them to their
                     * corresponding channel in batch and queue n-1 so that
                     * they can be handled by other handler threads. The last
                     * event is handled by this thread (and so is not queued).
                     */
                    fdToChannelLock.readLock().lock();
                    try {
                        while (n-- > 0) {
                            long eventAddress = getEvent(address, n);
                            int fd = getDescriptor(eventAddress);

                            // 。。。 省略
							
                            PollableChannel channel = fdToChannel.get(fd);
                            if (channel != null) {
                                int events = getEvents(eventAddress);
                                Event ev = new Event(channel, events);

                                // n-1 events are queued; This thread handles
                                // the last one except for the wakeup
                                if (n > 0) {
                                    queue.offer(ev);
                                } else {
                                    return ev;
                                }
                            }
                        }
                    } finally {
                        fdToChannelLock.readLock().unlock();
                    }
                }
            } finally {
                // to ensure that some thread will poll when all events have
                // been consumed
                queue.offer(NEED_TO_POLL);
            }
        }

        public void run() {
            Invoker.GroupAndInvokeCount myGroupAndInvokeCount =
                Invoker.getGroupAndInvokeCount();
            final boolean isPooledThread = (myGroupAndInvokeCount != null);
            boolean replaceMe = false;
            Event ev;
            try {
                for (;;) {
                    // reset invoke count
                    if (isPooledThread)
                        myGroupAndInvokeCount.resetInvokeCount();

                    try {
                        replaceMe = false;
                        ev = queue.take();

                        // no events and this thread has been "selected" to
                        // poll for more.
                        if (ev == NEED_TO_POLL) {
                            try {
                                ev = poll();
                            } catch (IOException x) {
                                x.printStackTrace();
                                return;
                            }
                        }
                    } catch (InterruptedException x) {
                        continue;
                    }

                    // 。。。 省略

                    // process event
                    try {
                        ev.channel().onEvent(ev.events(), isPooledThread);
                    } catch (Error x) {
                        replaceMe = true; throw x;
                    } catch (RuntimeException x) {
                        replaceMe = true; throw x;
                    }
                }
            } finally {
                // last handler to exit when shutdown releases resources
                int remaining = threadExit(this, replaceMe);
                if (remaining == 0 && isShutdown()) {
                    implClose();
                }
            }
        }
    }
}
```
上面方法的逻辑是：
* 在 EPollPort 启动时，会在线程池执行`EventHandlerTask`，这个Task主要做2个事情：
  * 从 Epoll 获取就绪事件
  * 分发就绪事件到相关Channel
* 通过`epollWait()`方法获取就绪事件，封装成Event
* 异步Channel的实现类，需要实现`Port.PollableChannel`接口，这个接口只有一个`onEvent()`方法，处理就绪事件的逻辑就在这里


#### 处理就绪事件
假设有数据可读，那么`UnixAsynchronousSocketChannelImpl`里面的`onEvent()`方法会被执行，然后调用`finishRead()`方法：
```
private void finishRead(boolean mayInvokeDirect) {
    int n = -1;
    Throwable exc = null;

    // copy fields as we can't access them after reading is re-enabled.
    boolean scattering = isScatteringRead;
    CompletionHandler<Number,Object> handler = readHandler;
    Object att = readAttachment;
    PendingFuture<Number,Object> future = readFuture;
    Future<?> timeout = readTimer;

    try {
        begin();

        if (scattering) {
            n = (int)IOUtil.read(fd, readBuffers, nd);
        } else {
            n = IOUtil.read(fd, readBuffer, -1, nd);
        }
        if (n == IOStatus.UNAVAILABLE) {
            // spurious wakeup, is this possible?
            synchronized (updateLock) {
                readPending = true;
            }
            return;
        }

        // allow objects to be GC'ed.
		/** 
		  * 留意：仅仅是这个Channel对象不引用 Buffer对象，
		  *       此时Buffer还不会被GC，因为用户代码有它的强引用
         */		  
        this.readBuffer = null;
        this.readBuffers = null;
        this.readAttachment = null;

        // allow another read to be initiated
        enableReading();

    } catch (Throwable x) {
        enableReading();
        if (x instanceof ClosedChannelException)
            x = new AsynchronousCloseException();
        exc = x;
    } finally {
        // restart poll in case of concurrent write
        if (!(exc instanceof AsynchronousCloseException))
            lockAndUpdateEvents();
        end();
    }

    // cancel the associated timer
    if (timeout != null)
        timeout.cancel(false);

    // create result
    Number result = (exc != null) ? null : (scattering) ?
        (Number)Long.valueOf(n) : (Number)Integer.valueOf(n);

    // invoke handler or set result
    if (handler == null) {
        future.setResult(result, exc);
    } else {
        if (mayInvokeDirect) {
            Invoker.invokeUnchecked(handler, att, result, exc);
        } else {
            Invoker.invokeIndirectly(this, handler, att, result, exc);
        }
    }
}
```
上面代码，只关注主要流程：
* 通过`IOUtil.read()`将 fd 里面的数据读到 readBuffer里面
* 根据传入的参数 handler是否为 null，通知用户的方式也不一样：
  * handler为null：即`CompletionHandler`方式，调用用户定义的回调函数
  * handler不为null：即`Future`方式，将 result 设置到 future 对象里面
* `Invoker.invokeIndirectly`：通过 channel group 的线程池，调用 handler，代码如下：
  ```
  /**
   * Invokes the handler indirectly via the channel group's thread pool.
   */
  static <V,A> void invokeIndirectly(AsynchronousChannel channel,
                                     final CompletionHandler<V,? super A> handler,
                                     final A attachment,
                                     final V result,
                                     final Throwable exc)
  {
      try {
          ((Groupable)channel).group().executeOnPooledThread(new Runnable() {
              public void run() {
                  GroupAndInvokeCount thisGroupAndInvokeCount =
                      myGroupAndInvokeCount.get();
                  if (thisGroupAndInvokeCount != null)
                      thisGroupAndInvokeCount.setInvokeCount(1);
                  invokeUnchecked(handler, attachment, result, exc);
              }
          });
      } catch (RejectedExecutionException ree) {
          throw new ShutdownChannelGroupException();
      }
  }
  
  
  /**
   * Invoke handler without checking the thread identity or number of handlers
   * on the thread stack.
   */
  static <V,A> void invokeUnchecked(CompletionHandler<V,? super A> handler,
                                    A attachment,
                                    V value,
                                    Throwable exc)
  {
      if (exc == null) {
          handler.completed(value, attachment);
      } else {
          handler.failed(exc, attachment);
      }  
      // clear interrupt
      Thread.interrupted();
  }
  ```
  
  
#### 总结
<img src="/assets/blogImg/Java_NIO/AsynchronousChannel/1.png">


* 使用异步Channel的Read，用户提供3个对象：
  * Buffer：数据将会读取到这个缓冲区
  * CompletionHandler：回调函数，IO操作完成时被调用
  * ThreadPool：线程池，基本整个IO过程都由它的线程执行，包括：
    * 检测、处理就绪事件
    * 执行回调函数逻辑
* 本文分析的代码，使用 Linux 的 Epoll：
  * 创建异步Channel对象时，调用`epollCreate`
  * 使用`read()`函数时，调用`epollCtl()`
  * 执行`EventHandlerTask`的线程（称它为IO线程），通过`epollWait()`获取就绪事件
* IO线程会调用`onEvent()`方法，处理就绪事件，在这里就是有数据可读：
  * 从FD拷贝数据到Buffer
  * 通知用户层
* 通知用户层有2种方式，根据调用哪一个版本的`read()`函数而不同：
  * `CompletionHandler`方式：IO线程执行用户定义的`CompletionHandler`的逻辑
  * `Future`方式：将 result 设置到 future 对象里面

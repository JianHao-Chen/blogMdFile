---
layout: post
title: "AsynchronousChannel的介绍"
date: 2018-08-17 11:36
comments: false
tags: 
- Java
- NIO
categories:	
- 基础
- Java
---

对于异步IO操作的检测和控制，AsynchronousChannel提供2种方式：
* 通过返回一个`java.util.concurrent.Future`对象来实现
* 通过传递一个`java.nio.channels.CompletionHandler`来完成，它会定义在操作完毕后所执行的处理程序方法。


下面我们以`AsynchronousServerSocketChannel`的`accept()`进行分析。

<!--more-->

### 使用例子

#### Future方式
```
AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel
                    .open()
                    .bind(new InetSocketAddress("127.0.0.1", 8899));
                    
Future<AsynchronousSocketChannel> future = serverSocketChannel.accept();

//检查是否完成（如果底层的操作已经完成，则返回 true。 请注意，在这种情况下，完成可能意味着正常终止、异常或取消。）
future.isDone();

// 检查操作是否取消 （如果操作在正常完成之前被取消，则返回 true。）
future.isCancelled();

// 尝试取消操作
future.cancel(true);

//获取操作结果
AsynchronousSocketChannel client= future.get();
AsynchronousSocketChannel worker = future.get(10, TimeUnit.SECONDS);
```

#### CompletionHandler 方式
```
AsynchronousServerSocketChannel listener = AsynchronousServerSocketChannel.open().bind(null);
 
listener.accept(
  attachment, new CompletionHandler<AsynchronousSocketChannel, Object>() {
    public void completed(
      AsynchronousSocketChannel client, Object attachment) {
          // do whatever with client
      }
    public void failed(Throwable exc, Object attachment) {
          // handle failure
      }
  });
```


### 实现源码

#### AsynchronousServerSocketChannel 的创建

我们可以使用静态方法 open 创建一个 AsynchronousServerSocketChannel 对象，它的定义如下：
```
public static AsynchronousServerSocketChannel open(AsynchronousChannelGroup group) throws IOException
{
    AsynchronousChannelProvider provider = (group == null) ?
            AsynchronousChannelProvider.provider() : group.provider();
    return provider.openAsynchronousServerSocketChannel(group);
}
```
上面方法的逻辑是：
* 如果入参的 group 为 null，那么使用默认的（系统级别的）AsynchronousChannelProvider，否则使用入参的 group。
* 调用 AsynchronousChannelProvider 的 `openAsynchronousServerSocketChannel`方法

看到`openAsynchronousServerSocketChannel`的实现：
```
// 在Linux下的实现在 LinuxAsynchronousChannelProvider
@Override
public AsynchronousServerSocketChannel openAsynchronousServerSocketChannel(AsynchronousChannelGroup group)
    throws IOException
{
    return new UnixAsynchronousServerSocketChannelImpl(toPort(group));
}
```
继续看UnixAsynchronousServerSocketChannelImpl的构造方法：
```
UnixAsynchronousServerSocketChannelImpl(Port port)
        throws IOException
{
    super(port);
    try {
        IOUtil.configureBlocking(fd, false);
    } catch (IOException x) {
        nd.close(fd);  // prevent leak
        throw x;
    }
    this.port = port;
    this.fdVal = IOUtil.fdVal(fd);
    // add mapping from file descriptor to this channel
    port.register(fdVal, this);
}
```
其中，Port是AsynchronousChannelGroup的Unix实现。



#### AsynchronousServerSocketChannel 的 accept
先看接口：
```
public abstract <A> void accept(A attachment,
                                    CompletionHandler<AsynchronousSocketChannel,? super A> handler);
```

🦎 补充（CompletionHandler的定义）：
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
继续看回 accept 的实现（在 UnixAsynchronousServerSocketChannelImpl类）：
```
@Override
    Future<AsynchronousSocketChannel> implAccept(Object att,
        CompletionHandler<AsynchronousSocketChannel,Object> handler)
{
    
    // complete immediately if channel is closed
    if (!isOpen()) {
        Throwable e = new ClosedChannelException();
        if (handler == null) {
            return CompletedFuture.withFailure(e);
        } else {
            Invoker.invoke(this, handler, att, null, e);
            return null;
        }
    }
    
    if (localAddress == null)
        throw new NotYetBoundException();
        
    // cancel was invoked with pending accept so connection may have been
    // dropped.
    if (isAcceptKilled())
        throw new RuntimeException("Accept not allowed due cancellation");
    // check and set flag to prevent concurrent accepting
    if (!accepting.compareAndSet(false, true))
        throw new AcceptPendingException();
        
    
    // attempt accept
    FileDescriptor newfd = new FileDescriptor();
    InetSocketAddress[] isaa = new InetSocketAddress[1];
    Throwable exc = null;
    
    try {
    
        begin();
        
        int n = accept0(this.fd, newfd, isaa);
        if (n == IOStatus.UNAVAILABLE) {
        
            // ... 省略部分代码
            
            // register for connections
            port.startPoll(fdVal, Port.POLLIN);
            return result;
        }
    
    } catch (Throwable x) {
        // accept failed
        if (x instanceof ClosedChannelException)
            x = new AsynchronousCloseException();
        exc = x;
    } finally {
        end();
    }
    
    AsynchronousSocketChannel child = null;
    if (exc == null) {
        // connection accepted immediately
        try {
            child = finishAccept(newfd, isaa[0], null);
        } catch (Throwable x) {
            exc = x;
        }
    }
    // re-enable accepting before invoking handler
    enableAccept();
    
    if (handler == null) {  //  使用 Future 方式的 accept
        return CompletedFuture.withResult(child, exc);
    } else {
        Invoker.invokeIndirectly(this, handler, att, child, exc);
        return null;
    }
}
```
上面方法有3个点值得注意：
* 通过`port.startPoll`，等待 POLLIN 事件。
* `finishAccept`方法：根据得到的新的fd，InetSocketAddress 创建AsynchronousSocketChannel对象
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

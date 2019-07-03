---
layout: post
title: "Socket和SocketChannel读取数据的差别"
date: 2019-05-13 08:55
comments: false
tags: 
- NIO
- Java
categories:	
- NIO
---


***Socket*** 是`java.net`包下的类，***SocketChannel*** 是`java.nio.channels`包下的类。下面将分析它们读操作的细节。
<!--more-->

### Socket的读操作

平时我们会用到Socket的`getInputStream()`方法，它其实是取SocketImpl的`getInputStream()`，这里的InputStream就是`SocketInputStream`。
 
接下来自然是看`SocketInputStream`的`read`方法：
```
/**
 * Reads into a byte array <i>b</i> at offset <i>off</i>,
 * <i>length</i> bytes of data.
 * @param b the buffer into which the data is read
 * @param off the start offset of the data
 * @param length the maximum number of bytes read
 * @return the actual number of bytes read, -1 is
 *          returned when the end of the stream is reached.
 * @exception IOException If an I/O error has occurred.
 */
public int read(byte b[], int off, int length) throws IOException {
    return read(b, off, length, impl.getTimeout());
}


int read(byte b[], int off, int length, int timeout) throws IOException {
   int n;
   
   // EOF already encountered
   if (eof) {
       return -1;
   }

   // connection reset
   if (impl.isConnectionReset()) {
       throw new SocketException("Connection reset");
   }
   
   
   // bounds check
   if (length <= 0 || off < 0 || length > b.length - off) {
       if (length == 0) {
           return 0;
       }
       throw new ArrayIndexOutOfBoundsException("length == " + length
               + " off == " + off + " buffer length == " + b.length);
   }
   
   
   boolean gotReset = false;

   // acquire file descriptor and do the read
   FileDescriptor fd = impl.acquireFD();
   try {
       n = socketRead(fd, b, off, length, timeout);
       if (n > 0) {
           return n;
       }
   } catch (ConnectionResetException rstExc) {
       gotReset = true;
   } finally {
       impl.releaseFD();
   }
   
   
   /*
    * We receive a "connection reset" but there may be bytes still
    * buffered on the socket
    */
   if (gotReset) {
       impl.setConnectionResetPending();
       impl.acquireFD();
       try {
           n = socketRead(fd, b, off, length, timeout);
           if (n > 0) {
               return n;
           }
       } catch (ConnectionResetException rstExc) {
       } finally {
           impl.releaseFD();
       }
   }
   
   
   /*
    * If we get here we are at EOF, the socket has been closed,
    * or the connection has been reset.
    */
   if (impl.isClosedOrPending()) {
       throw new SocketException("Socket closed");
   }
   if (impl.isConnectionResetPending()) {
       impl.setConnectionReset();
   }
   if (impl.isConnectionReset()) {
       throw new SocketException("Connection reset");
   }
   eof = true;
   return -1;

}
```
这个方法的逻辑如下：
1. 先进行状态检查（EOF、connection reset），因为检查不通过，就没必要往下执行了。
2. 入参检查
3. 获取fd(计数加1)，调用native方法`socketRead(fd, byte[], off, len, timeout)`
4. 如果读取成功，那么返回读取字节数。
5. 如果catch到`ConnectionResetException`，这时候有2种可能：
  * 在底层socket有若干数据，于是重试第3步。如果读取成功，那么返回读取字节数。
  * 在底层socket已经没有数据了，此时会读到EOF，那么进入第6步
6. 设置SocketImpl的状态，如果socket已经close或者reset，那么抛出相应的Exception，否则返回-1表示读到EOF。

这里，需要看一下那个native的`socketRead`方法，在 *SocketInputStream.c* ：
```
JNIEXPORT jint JNICALL
Java_java_net_SocketInputStream_socketRead0(JNIEnv *env, jobject this,
                                            jobject fdObj, jbyteArray data,
                                            jint off, jint len, jint timeout)
{

    char BUF[MAX_BUFFER_LEN];
    char *bufP;
    jint fd, nread;
    
    if (IS_NULL(fdObj)) {
        JNU_ThrowByName(env, JNU_JAVANETPKG "SocketException","Socket closed");
        return -1;
    }
    else {
        fd = (*env)->GetIntField(env, fdObj, IO_fd_fdID);
        if (fd == -1) {
            JNU_ThrowByName(env, "java/net/SocketException", "Socket closed");
            return -1;
        }
    }
    
    
    /*
     * If the read is greater than our stack allocated buffer then
     * we allocate from the heap (up to a limit)
     */
    if (len > MAX_BUFFER_LEN) {
        if (len > MAX_HEAP_BUFFER_LEN) {
            len = MAX_HEAP_BUFFER_LEN;
        }
        bufP = (char *)malloc((size_t)len);
        if (bufP == NULL) {
            bufP = BUF;
            len = MAX_BUFFER_LEN;
        }
    } else {
        bufP = BUF;
    }

    
    if (timeout) {
        nread = NET_Timeout(fd, timeout);
        if (nread <= 0) {
            if (nread == 0) {
                JNU_ThrowByName(env, JNU_JAVANETPKG "SocketTimeoutException","Read timed out");
            } 
            else if (nread == JVM_IO_ERR) {
                if (errno == EBADF) {
                     JNU_ThrowByName(env, JNU_JAVANETPKG "SocketException","Socket closed");
                 } 
                 else {
                     NET_ThrowByNameWithLastError(env, JNU_JAVANETPKG "SocketException","select/poll failed");
                 }
            } 
            else if (nread == JVM_IO_INTR) {
                JNU_ThrowByName(env, JNU_JAVAIOPKG "InterruptedIOException","Operation interrupted");
            }
            if (bufP != BUF) {
                free(bufP);
            }
            return -1;
        }
    }
    
    
    nread = NET_Read(fd, bufP, len);
    if (nread <= 0) {
        if (nread < 0) {

            switch (errno) {
                case ECONNRESET:
                case EPIPE:
                    JNU_ThrowByName(env, "sun/net/ConnectionResetException",
                        "Connection reset");
                    break;

                case EBADF:
                    JNU_ThrowByName(env, JNU_JAVANETPKG "SocketException",
                        "Socket closed");
                    break;

                case EINTR:
                     JNU_ThrowByName(env, JNU_JAVAIOPKG "InterruptedIOException",
                           "Operation interrupted");
                     break;

                default:
                    NET_ThrowByNameWithLastError(env,
                        JNU_JAVANETPKG "SocketException", "Read failed");
            }
        }
    }
    else {
        (*env)->SetByteArrayRegion(env, data, off, nread, (jbyte *)bufP);
    }
    
    if (bufP != BUF) {
        free(bufP);
    }
    return nread;
}
```
上面的代码主要分为4步：
* 检查：包括对Java对象FileDescriptor的检查，fd的检查
* 对读取数据字节数的限制：
  * 如果 len 超过 MAX_BUFFER_LEN， 那么需要在JNI环境使用`malloc`申请堆内存
  * 如果 len 超过 MAX_HEAP_BUFFER_LEN，len 最大也只能为 MAX_HEAP_BUFFER_LEN。
* 如果入参的timeout>0，那么使用`NET_Timeout`方法，这个方法在linux_close.c里面，它就是对`poll`方法的一层包装，实现有超时的轮询。它的结果nread有2种情况：
  * nread<=0，抛出异常（如SocketTimeoutException、Socket closed、select/poll failed、InterruptedIOException）
  * nread>0，就是`poll`的返回值>0，表示监听的fd有数据，进入第3步
* 调用`NET_Read`方法（在linux_close.c里面），它是对系统调用`ssize_t recv(int sockfd, void *buf, size_t len, int flags);`做了封装，对于被中断的情况，自动重新开始。同样的，它的结果nread有2种情况：
  * nread <= 0，抛出异常（如ConnectionResetException、Socket closed、InterruptedIOException）
  * nread > 0，表示已经成功读取数据到数组（此时指的是在JNI里面的数组），然后通过`JNIEnv->SetByteArrayRegion()`方法将数据拷贝到Java heap的数组。
 

<font size=4>**小结**</font>
* socket（SocketInputStream）的读操作，本质是依靠 **系统调用recv**。因此，数据需要先从内核缓冲区拷贝到JVM层的缓冲区，然后再拷贝到Java应用的缓冲区。
* 在使用系统调用recv时，用到的缓冲区大小最大为`MAX_HEAP_BUFFER_LEN`，意味着如果调用此JNI方法的Java线程，如果希望读取的字节数大于`MAX_HEAP_BUFFER_LEN`，那么就需要调用多次（数据拷贝当然也是多次）。
* Socket的流（上面的SocketInputStream）：
  * 是单向的，它只有read方法，没有write方法。
  * <font color=red>它的read操作是JNI调用，并且是阻塞，只有在read方法读到数据或socket关闭、reset等异常，或超时，才会返回。</font>
  


### SocketChannel的读操作
使用SocketChannel读取数据，需要与Buffer配合使用：
```
// 创建了一个ByteBuffer，将从SocketChannel里面读取数据到这个ByteBuffer。
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = socketChannel.read(buf);
```
从`SocketChannel::read`开始，每一层的调用如下：
1. `SocketChannelImpl::read(ByteBuffer buf)`
2. `IOUtil::read(FileDescriptor fd, ByteBuffer dst, long position,NativeDispatcher nd, Object lock)`
3. `SocketDispatcher::read(FileDescriptor fd, long address, int len)`
4. `FileDispatcherImpl::read0(FileDescriptor fd, long address, int len)`
5. JNI方法（在FileDispatcherImpl.c）的`read0(JNIEnv *env, jclass clazz, jobject fdo, jlong address, jint len)`
6. 系统调用`ssize_t read(int fd, void *buf, size_t count);`


<font size=4>**小结**</font>
* SocketChannel的读操作，本质是依靠***系统调用read***。数据需要先从内核缓冲区拷贝到JVM层的缓冲区，然后再拷贝到Java应用的缓冲区。
* 在`IOUtil::read`方法里，是使用直接缓冲区(DirectBuffer)的，如果Java应用线程调用时传入的是heap buffer，那么还会导致一次拷贝（原因在《Java堆外内存》那一篇讲述）。



### 总结
* 从SocketChannel还是SocketInputStream读数据，都需要先从内核缓冲区拷贝到JVM层的缓冲区，然后再拷贝到Java应用的缓冲区。
* 如果读取大量的数据：
  * 使用SocketInputStream将会导致多次的native方法的调用（由于存在缓冲区大小限制）。
  * 使用SocketChannel，只需要一次native方法的调用（前提是提供的缓冲区足够）。
* 关于阻塞：
  * SocketInputStream的read是阻塞的，直到数据已经读取到Java应用的缓冲区（正常流程）
  * SocketChannel提供非阻塞模式



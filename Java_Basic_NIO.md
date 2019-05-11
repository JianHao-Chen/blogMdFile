---
layout: post
title: "Java NIO 简介"
date: 2018-04-01 08:55
comments: false
tags: 
- IO
- Java
categories:	
- 基础
- Java
---

### 介绍NIO
NIO包（java.nio.*）引入了四个关键的抽象数据类型，它们共同解决传统的I/O类中的一些问题。

1. Buffer：它是包含数据且用于读写的线形表结构。其中还提供了一个特殊类用于内存映射文件的I/O操作。
2. Charset：它提供Unicode字符串影射到字节序列以及逆影射的操作。
3. Channels：包含socket，file和pipe三种管道，它实际上是双向交流的通道。
4. Selector：它将多元异步I/O操作集中到一个或多个线程中（它可以被看成是Unix中select（）函数或Win32中WaitForSingleEvent（）函数的面向对象版本）。

<!--more-->

#### 缓冲区(Buffer)
缓冲区主要用来作为从通道发送或者接收数据的容器。缓冲区实质上是一个数组。通常它是一个字节数组，但是也可以使用其他种类的数组。但是一个缓冲区不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。 

对于每一种基本 Java 类型都有一种缓冲区类型： 

* ByteBuffer
* CharBuffer
* ShortBuffer
* IntBuffer
* LongBuffer
* FloatBuffer
* DoubleBuffer

#### 字符编码(Charset)
向ByteBuffer中存放数据涉及到两个问题：字节的顺序和字符转换。ByteBuffer内部通过ByteOrder类处理了字节顺序问题，但是并没有处理字符转换。事实上，ByteBuffer没有提供方法读写String。
Java.nio.charset.Charset处理了字符转换问题。它通过构造CharsetEncoder和CharsetDecoder将字符序列转换成字节和逆转换。

#### 通道(Channel)
Channel用于在字节缓冲区和位于通道另一侧的实体（通常是一个文件或套接字）之间有效地传输数据。


#### 选择器(Selector)
选择器提供选择执行已经就绪的任务的能力。在过去的阻塞I/O中，我们一般知道什么时候可以向stream中读或写，因为方法调用直到stream准备好时返回。但是使用非阻塞通道，我们需要一些方法来知道什么时候通道准备好了。在NIO包中，设计Selector就是为了这个目的。


### NIO的新特性

#### 多路复用I/O
I/O多路复用这种I/O模型就不多解释了。
在Linux的系统调用中，有select/poll，还有epoll。(Windows的相关系统调用不清楚)，它们都用于帮我们侦测多个fd是否就绪。select/poll是顺序扫描fd是否就绪，而且支持的fd数量有限。epoll是基于事件驱动方式，而不是顺序扫描,当有fd就绪时，立即回调函数rollback。

不过nio中的Selector的取名总让我以为是类似于select/poll的模型，但是你会发现，当有数据被准备好时，调用完select()后，会返回一个SelectionKey，SelectionKey表示在某个selector上的某个Channel的数据已经被准备好了。那到底底层实现是使用select/poll还是epoll呢？

```bash
public static SelectorProvider create() {
PrivilegedAction pa = new GetPropertyAction("os.name");
  String osname = (String) AccessController.doPrivileged(pa);
  if ("SunOS".equals(osname)) {
    return new sun.nio.ch.DevPollSelectorProvider();
  }
 
  // use EPollSelectorProvider for Linux kernels >= 2.6
  if ("Linux".equals(osname)) {
    pa = new GetPropertyAction("os.version");
    String osversion = (String) AccessController.doPrivileged(pa);
    String[] vers = osversion.split("\\.", 0);
    if (vers.length >= 2) {
      try {
        int major = Integer.parseInt(vers[0]);
        int minor = Integer.parseInt(vers[1]);
        if (major > 2 || (major == 2 && minor >= 6)) {
          return new sun.nio.ch.EPollSelectorProvider();
        }
      } catch (NumberFormatException x) {
        // format not recognized
      }
    }
  }
  return new sun.nio.ch.PollSelectorProvider();
}
```
**可以看到在Linux下，内核版本大于2.6时使用epoll，小于2.6时使用poll。**

#### 非阻塞套接字
传统的Java I/O模型缺少非阻塞I/O从一开始就已经惹眼了，最后终于有了NIO。从SelectableChannel继承的Channel类可以用configureBlocking替换成为非阻塞模式。在J2SE1.4中只有套接字通道（SocketChannel， ServerSocketChannel和DatagramChannel）可以换为非阻塞模式，而FileChannel不能设为非阻塞模式。

当通道设置为非阻塞时，read()或者write()调用不管有没有传输数据总是立即返回。这样线程总是可以无停滞地检查数据是否已经准备好。

#### 直接通道传输

使用场景：
我们读一块数据到缓冲区，然后立即写到某个地方，中间没有用户的操作。

现在，有了下面的API，我们可以使数据被读取到目的地，而不经过中间的用户缓冲区。
```bash
public abstract class FileChannel
  extends AbstractChannel
  implements ByteChannel, GatheringByteChannel, ScatteringByteChannel{
  
  // This is a partial API listing
 
  public abstract long transferTo (long position, long count, 
    WritableByteChannel target)
 
  public abstract long transferFrom (ReadableByteChannel src,  
    long position, long count)
}
```
基于操作系统提供的支持，整个通道传输能够在内核中完成。这不仅缓解了繁重的拷贝工作，而且绕过了JVM！

#### 分散读和聚集写
分散/聚集 I/O 是使用多个而不是单个缓冲区来保存数据的读写方法。

通道可以有选择地实现两个新的接口： ScatteringByteChannel 和 GatheringByteChannel。

一个 ScatteringByteChannel 是一个具有两个附加读方法的通道： 
```bash
long read( ByteBuffer[] dsts );
long read( ByteBuffer[] dsts, int offset, int length );
```
这些 long read() 方法很像标准的 read 方法，只不过它们不是取单个缓冲区而是取一个缓冲区数组。

在 分散读取 中，通道依次填充每个缓冲区。填满一个缓冲区后，它就开始填充下一个。在某种意义上，缓冲区数组就像一个大缓冲区。 

GatheringByteChannel 是一个具有两个附加写方法的通道：
```bash
long write( ByteBuffer[] srcs );
long write( ByteBuffer[] srcs, int offset, int length );
```

一个分散的读取就像一个常规通道读取，只不过它是将数据读到一个缓冲区数组中而不是读到单个缓冲区中。同样地，一个聚集写入是向缓冲区数组而不是向单个缓冲区写入数据。 

使用场景：您可能在编写一个使用消息对象的网络应用程序，每一个消息被划分为固定长度的头部和固定长度的正文。您可以创建一个刚好可以容纳头部的缓冲区和另一个刚好可以容难正文的缓冲区。当您将它们放入一个数组中并使用分散读取来向它们读入消息时，头部和正文将整齐地划分到这两个缓冲区中。 


#### 内存映射文件
让我们用一种特殊的ByteBuffer——MappedByteBuffer来继续讨论用任意内存空间包装ByteBuffer对象的主题。在大多数操作系统上，可以通过mmap系统调用（或者相似的操作）在一个已打开的文件描述符上做内存映射文件。调用mmap返回一个指向内存段的指针，实际上代表文件的内容。从内存区域获取数据实际上就是从相应文件的偏移位置处返回数据。而修改则会将文件从内存空间写入磁盘。

【补充】我对内存映射文件的理解：
首先，“映射”这个词，就和数学课上说的“一一映射”是一个意思，就是建立一种一一对应关系，在这里主要是只硬盘上文件的位置与进程逻辑地址空间中一块大小相同的区域之间的一一对应，如下图所示。
<img width=800 src="/assets/blogImg/Java_Basic/NIO/1.png">

这种对应关系纯属是逻辑上的概念，物理上是不存在的，原因是进程的逻辑地址空间本身就是不存在的。在内存映射的过程中，并没有实际的数据拷贝，文件没有被载入内存，只是逻辑上被放入了内存，具体到代码，就是建立并初始化了相关的数据结构（struct address_space），这个过程有系统调用mmap()实现，所以建立内存映射的效率很高。

<font color=red>既然建立内存映射没有进行实际的数据拷贝，那么进程又怎么能最终直接通过内存操作访问到硬盘上的文件呢？</font>

mmap()会返回一个指针ptr，它指向进程逻辑地址空间中的一个地址，这样以后，进程无需再调用read或write对文件进行读写，而只需要通过ptr就能够操作文件。但是ptr所指向的是一个逻辑地址，要操作其中的数据，必须通过MMU将逻辑地址转换成物理地址，如过程2所示。这个过程与内存映射无关。

MMU在地址映射表中是无法找到与ptr相对应的物理地址的，也就是MMU失败，将产生一个缺页中断，缺页中断的中断响应函数会在swap中寻找相对应的页面，如果找不到（也就是该文件从来没有被读入内存的情况），则会通过mmap()建立的映射关系，从硬盘上将文件读取到物理内存中，如过程3所示。这个过程与内存映射无关。

如果在拷贝数据时，发现物理内存不够用，则会通过虚拟内存机制（swap）将暂时不用的物理页面交换到硬盘上，如过程4所示。这个过程也与内存映射无关。

<font color=red>**效率问题**</font>
从硬盘上将文件读入内存，都要经过文件系统进行数据拷贝，并且数据拷贝操作是由文件系统和硬件驱动实现的，理论上来说，拷贝数据的效率是一样的。但是通过内存映射的方法访问硬盘上的文件，效率要比read和write系统调用高，这是为什么呢？

原因是read()是系统调用，其中进行了数据拷贝，它首先将文件内容从硬盘拷贝到内核空间的一个缓冲区，如过程1，然后再将这些数据拷贝到用户空间，如过程2，在这个过程中，实际上完成了两次数据拷贝；而mmap()也是系统调用，如前所述，mmap()中没有进行数据拷贝，真正的数据拷贝是在缺页中断处理时进行的，由于mmap()将文件直接映射到用户空间，所以中断处理函数根据这个映射关系，直接将文件从硬盘拷贝到用户空间，只进行了一次数据拷贝 。因此，内存映射的效率要比read/write效率高。 
<img width=700 src="/assets/blogImg/Java_Basic/NIO/2.png">


#### 直接缓冲区
Sun 的文档是这样描述直接缓冲区的： 
给定一个直接字节缓冲区，Java虚拟机将尽最大努力直接对它执行本机I/O操作。也就是说，它会在每一次调用底层操作系统的本机 I/O 操作之前(或之后)，尝试避免将缓冲区的内容拷贝到一个中间缓冲区中(或者从一个中间缓冲区中拷贝数据)。

当你调用ByteBuffer.allocateDirect()创建一个直接缓冲区时，会分配本地系统内存（JVM内存堆以外的本地内存空间）并且用一个缓冲区对象来包装它。

现在，JNI代码能够定位Java使用ByteBuffer.allocateDirect()创建的本地内存空间地址，而且它还能够分配内存（例如：使用malloc）并且通过回调在JVM中把这个内存空间包装成新的ByteBuffer对象（JNI中方法是NewDirectByteBuffer()）。

让人真正兴奋的地方是ByteBuffer对象能够包装任何本地代码的内存地址，甚至JVM以外的地址空间。一个例子是创建直接ByteBuffer对象来封装显卡的内存。这样的缓冲区允许纯Java代码直接读写显卡内存，不用做系统调用或者缓冲区拷贝。完全用Java编写显示驱动！你需要的仅仅是使用JNI来访问显示内存并且返回一个ByteBuffer对象。在NIO之前这是不可能完成的。


#### 缓冲区视图
缓冲区主要用来作为从通道发送或者接收数据的容器。通道是低级I/O服务的管道，他们是面向字节的；所以他们只能操作ByteBuffer对象。

那么我们用其他缓冲区类型（指的是CharBuffer,DoubleBuffer,IntBuffer等）干什么呢？

例如，假设你有一个存储16比特Unicode（这里指的是UTF-16编码，不是普通文件中使用的UTF-8编码）字符的文件。如果你读了一块文件到字节缓冲区内，你可以像这样创建它们的字符缓冲区视图。

```bash
CharBuffer charBuffer = byteBuffer.asCharBuffer();
```
上面这段代码创建了一个带有CharBuffer行为的ByteBuffer视图。如下图所示，缓冲区中的每对字节组成一个16bit char字符。
<img width=500 src="/assets/blogImg/Java_Basic/NIO/3.png">
接着你可以用CharBuffer对象在数据上迭代（用相对get()方法），用绝对get()方法随机访问，或者将数据拷贝到char数组并把它传递给一个与缓冲无关的对象。

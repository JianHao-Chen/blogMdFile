---
layout: post
title: "Java堆外内存"
date: 2019-05-14 08:55
comments: false
tags: 
- NIO
- Java
categories:	
- NIO
---

本文分析堆外内存的原理，包括以下部分：
* 堆外内存的创建
* 堆外内存的回收
* 使用堆外内存读写
* 堆外内存的优势

<!--more-->

### 堆外内存的创建

```
// Primary constructor
DirectByteBuffer(int cap) {                 // package-private
    super(-1, 0, cap, cap);                 // 设置 mark, position, limit, and capacity
    boolean pa = VM.isDirectMemoryPageAligned();    // 和内存对齐相关
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    
    /**
    *   这里并没有实际申请内存，只是想 Bits类 申请。
    */
    Bits.reserveMemory(size, cap);
    
    /**
    *  这里是真正的申请内存
    */
    long base = 0;
    try {
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    
    /**
    *   对内存清空、对齐
    */
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

#### Bits的reserveMemory方法

Bits类保存了下面的变量，用于管理直接内存的分配。
```
private static final AtomicLong reservedMemory = new AtomicLong();
private static final AtomicLong totalCapacity = new AtomicLong();
private static final AtomicLong count = new AtomicLong();
```

当向Bits申请内存，其实只是改变它的变量的值。如果超过限定的值，会执行 `System.gc()`，期待能主动回收一点堆外内存。如果内存还是不足，就抛出OOM异常。


#### Unsafe的allocateMemory
Unsafe的allocateMemory是一个native方法，它的C++代码如下：
```
UNSAFE_ENTRY(jlong, Unsafe_AllocateMemory(JNIEnv *env, jobject unsafe, jlong size))
  UnsafeWrapper("Unsafe_AllocateMemory");
  size_t sz = (size_t)size;
  if (sz != (julong)size || size < 0) {
    THROW_0(vmSymbols::java_lang_IllegalArgumentException());
  }
  if (sz == 0) {
    return 0;
  }
  sz = round_to(sz, HeapWordSize);
  void* x = os::malloc(sz);
  if (x == NULL) {
    THROW_0(vmSymbols::java_lang_OutOfMemoryError());
  }
  //Copy::fill_to_words((HeapWord*)x, sz / HeapWordSize);
  return addr_to_java(x);
UNSAFE_END
```
可以看到，使用的是 malloc 分配内存。

#### 创建并赋值 cleaner成员变量
之所以需要这个 Cleaner类，是和直接内存的释放有关的。



### 堆外内存的回收

申请了堆外内存以后，在Java Heap里，有DirectByteBuffer对象，同时，在Heap外，有一段内存，它的地址就保存在DirectByteBuffer对象的address成员变量中。这就是所谓的***冰山对象***。
显然，内存回收就包括2部分了。

#### DirectByteBuffer对象的内存
这里说的就是DirectByteBuffer对象在Java Heap的那一部分，它的回收是符合GC机制的。

#### 直接内存的回收
这里说的回收指的是依靠JVM，而不是程序主动的调用如Unsafe的freeMemory之类的方法。

<font color=orange>**也就是当DirectByteBuffer对象被GC掉了，它背后的堆外内存也会被释放掉！**</font>

#### Cleaner类
```
public class Cleaner extends PhantomReference<Object> {
  private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue();
  private static Cleaner first = null;
  private Cleaner next = null;
  private Cleaner prev = null;
  private final Runnable thunk;
  
  ...

  public void clean() {
    if (remove(this)) {
      try {
        this.thunk.run();
      } catch (final Throwable var2) {
         // ...
      }

    }
  }
  
}
```

Cleaner类是PhantomReference（虚引用），虚引用的机制是：
* 如果一个对象只有虚引用指向它，那么这个对象是可以被GC的。
* 当一个虚引用指向的对象被GC了，这个虚引用也会被放到一个队列里面


#### ReferenceHandler
<font color=orange>ReferenceHandler线程，关注着前面提到的队列，如果看到有对象类型是Cleaner，就会执行它的`clean()`方法。</font>
可以看到，Cleaner的clean方法，还是调用thunk（Runnable类型）的run方法，在我们关心的DirectByteBuffer中，就是定义在它内部的静态类 Deallocator。


#### Deallocator
```
private static class Deallocator implements Runnable{

    private static Unsafe unsafe = Unsafe.getUnsafe();

    private long address;
    private long size;
    private int capacity;

    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }

    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }

}
```
可以看到：
* Deallocator保存了DirectByteBuffer中堆外内存的地址address，还有用于Bits的size和capacity。
* 它的run方法做的就是：
    * 调用 Unsafe的freeMemory方法
    * 向Bits交还“额度”



### 使用DirectByteBuffer读写

#### 写操作
在Java的`SocketChannel`里面的write方法：
```
/**
*   Writes a sequence of bytes to this channel from the given buffer.
*/
public abstract int write(ByteBuffer src) throws IOException;
```
它的其中一个实现在 `sun.nio.ch.SocketChannelImpl`：
```
public int write(ByteBuffer buf) throws IOException {
    // ... 省略 
    
    try{
        // ... 省略
        for (;;) {
            n = IOUtil.write(fd, buf, -1, nd, writeLock);
            if ((n == IOStatus.INTERRUPTED) && isOpen())
                continue;
            return IOStatus.normalize(n);
    }
    finally{
        // ....省略
    }
}
```
继续看`IOUtil`的write方法：
```
static int write(FileDescriptor fd, ByteBuffer src, long position,
                NativeDispatcher nd, Object lock)throws IOException {
                
    if (src instanceof DirectBuffer)
            return writeFromNativeBuffer(fd, src, position, nd, lock);

    // Substitute a native buffer
    int pos = src.position();
    int lim = src.limit();
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);
    ByteBuffer bb = Util.getTemporaryDirectBuffer(rem);
    try {
        bb.put(src);
        bb.flip();
        // Do not update src until we see how many bytes were written
        src.position(pos);

        int n = writeFromNativeBuffer(fd, bb, position, nd, lock);
        if (n > 0) {
            // now update src
            src.position(pos + n);
        }
        return n;
    } finally {
        Util.offerFirstTemporaryDirectBuffer(bb);
    }
}
```
上面方法的逻辑是：
* 如果这是一个DirectBuffer，那么进入`writeFromNativeBuffer`方法
* 否则，创建一个临时的DirectBuffer，把当前ByteBuffer的数据拷贝到临时buffer里面，再调用`writeFromNativeBuffer`方法。


再看看`writeFromNativeBuffer`方法：
```
private static int writeFromNativeBuffer(FileDescriptor fd, ByteBuffer bb,
            long position, NativeDispatcher nd, Object lock) throws IOException {
            
    int pos = bb.position();
    int lim = bb.limit();
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);

    int written = 0;
    if (rem == 0)
        return 0;
    if (position != -1) {
        written = nd.pwrite(fd,
                            ((DirectBuffer)bb).address() + pos,
                            rem, position, lock);
    } else {
        written = nd.write(fd, ((DirectBuffer)bb).address() + pos, rem);
    }
    if (written > 0)
        bb.position(pos + written);
    return written;
}
```


#### 读操作
在Java的`SocketChannel`里面的read方法：
```
/**
 * Reads a sequence of bytes from this channel into the given buffer.
*/
public int read(ByteBuffer dst) throws IOException;
```
同样的，可以在IOUtil找到对应的`read`方法：
```
static int read(FileDescriptor fd, ByteBuffer dst, long position,
                    NativeDispatcher nd, Object lock) throws IOException {
                    
    if (dst.isReadOnly())
        throw new IllegalArgumentException("Read-only buffer");
    if (dst instanceof DirectBuffer)
        return readIntoNativeBuffer(fd, dst, position, nd, lock);

    // Substitute a native buffer
    ByteBuffer bb = Util.getTemporaryDirectBuffer(dst.remaining());
    try {
        int n = readIntoNativeBuffer(fd, bb, position, nd, lock);
        bb.flip();
        if (n > 0)
            dst.put(bb);
        return n;
    } finally {
        Util.offerFirstTemporaryDirectBuffer(bb);
    }
}
```
上面方法的逻辑是：
* 如果这是一个DirectBuffer，那么进入`readIntoNativeBuffer`方法
* 否则，创建一个临时的DirectBuffer，调用`readIntoNativeBuffer`，把通过fd读取到数据放在临时的DirectBuffer,然后再把临时数据拷贝到用户指定的ByteBuffer。


### 总结

执行网络IO或者文件IO时，对于非DirectBuffer的，JDK会创建一个临时DirectBuffer，然后再进行IO操作。

<font size=4 color=green>原因:</font>
当我们把一个地址通过JNI传递到c库时，有一个基本要求是<font color=purple>**这个地址不能失效！**</font>但是，在GC管理机制下，Java堆里面的对象的地址是可能改变的，因此，必须把这个buffer放在GC管不到的地方，所以，需要DirectBuffer。

所以，可以说<font color=pink>***使用DirectBuffer进行IO的好处是减少1次的内存拷贝***</font>。
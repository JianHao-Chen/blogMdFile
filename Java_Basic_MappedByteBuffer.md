---
layout: post
title: "Java的文件映射"
date: 2018-08-03 20:36
comments: false
tags: 
- Java
- NIO
categories:	
- 基础
- Java
---

这篇文章介绍Java的文件映射。


<!--more-->

#### MappedByteBuffer的创建
在`FileChannel`有map方法用于“映射”文件的内容到内存：
```
public abstract MappedByteBuffer map(MapMode mode,
                long position, long size) throws IOException;
```
这里引用一下它的说明：

> Maps a region of this channel's file directly into memory.
> 🌟 A region of a file may be mapped into memory in one of three modes:
> * Read-only : Any attempt to modify the resulting buffer will cause a `java.nio.ReadOnlyBufferException`to be thrown.
> * Read/write : Changes made to the resulting buffer will eventually be propagated to the file; they may or may not be made visible to other programs that have mapped the same file.
> * Private : Changes made to the resulting buffer will not be propagated to the file and will not be visible to other programs that have mapped the same file; instead, they will cause private copies of the modified portions of the buffer to be created.
>
> 🌟 For a read-only mapping, this channel must have been opened for reading; for a read/write or private mapping, this channel must have been opened for both reading and writing.
> 
> 🌟 The MappedByteBuffer mapped byte buffer returned by this method will have a position of zero and a limit and capacity of size its mark will be undefined.  The buffer and the mapping that it represents will remain valid until the buffer itself is garbage-collected.
>
> 🌟 A mapping, once established, is not dependent upon the file channel that was used to create it. Closing the channel, in particular, has no effect upon the validity of the mapping.
>
> 🌟 For most operating systems, mapping a file into memory is more expensive than reading or writing a few tens of kilobytes of data via the usual read and write methods. From the standpoint of performance it is generally only worth mapping relatively large files into memory.


看一下它的实现,在`FileChannelImpl`：
```
public MappedByteBuffer map(MapMode mode, long position, long size) throws IOException {

    //...省略
    
    if (!isOpen())
        return null;
    if (size() < position + size) { // Extend file size
        if (!writable) {
            throw new IOException("Channel not open for writing " +
                "- cannot extend file to required size");
        }
        int rv;
        do {
            rv = nd.truncate(fd, position + size);
        } while ((rv == IOStatus.INTERRUPTED) && isOpen());
    }
    if (size == 0) {
        addr = 0;
        // a valid file descriptor is not required
        FileDescriptor dummy = new FileDescriptor();
        if ((!writable) || (imode == MAP_RO))
            return Util.newMappedByteBufferR(0, 0, dummy, null);
        else
            return Util.newMappedByteBuffer(0, 0, dummy, null);
    }
    
    
    int pagePosition = (int)(position % allocationGranularity);
    long mapPosition = position - pagePosition;
    long mapSize = size + pagePosition;
        
        
    try {
        // If no exception was thrown from map0, the address is valid
        addr = map0(imode, mapPosition, mapSize);
    } catch (OutOfMemoryError x) {
        // An OutOfMemoryError may indicate that we've exhausted memory
        // so force gc and re-attempt map
        // ...省略
    }
    
    
    // On Windows, and potentially other platforms, we need an open
    // file descriptor for some mapping operations.
    FileDescriptor mfd;
    try {
        mfd = nd.duplicateForMapping(fd);
    } catch (IOException ioe) {
        unmap0(addr, mapSize);
        throw ioe;
    }
            
        
    assert (IOStatus.checkAll(addr));
    assert (addr % allocationGranularity == 0);
    int isize = (int)size;
    Unmapper um = new Unmapper(addr, mapSize, isize, mfd);
    if ((!writable) || (imode == MAP_RO)) {
        return Util.newMappedByteBufferR(isize,
                                         addr + pagePosition,
                                         mfd,
                                         um);
    } else {
        return Util.newMappedByteBuffer(isize,
                                        addr + pagePosition,
                                        mfd,
                                        um);
    }

}
```

先看它的`map0`方法，是一个native方法，找到FileChannelImpl.c里面的实现：
```
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_map0(JNIEnv *env, jobject this,
                                     jint prot, jlong off, jlong len)
{
    void *mapAddress = 0;
    jobject fdo = (*env)->GetObjectField(env, this, chan_fd);
    jint fd = fdval(env, fdo);
    int protections = 0;
    int flags = 0;

    if (prot == sun_nio_ch_FileChannelImpl_MAP_RO) {
        protections = PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_RW) {
        protections = PROT_WRITE | PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_PV) {
        protections =  PROT_WRITE | PROT_READ;
        flags = MAP_PRIVATE;
    }

    mapAddress = mmap64(
        0,                    /* Let OS decide location */
        len,                  /* Number of bytes to map */
        protections,          /* File permissions */
        flags,                /* Changes are shared */
        fd,                   /* File descriptor of mapped file */
        off);                 /* Offset into file */

    if (mapAddress == MAP_FAILED) {
        if (errno == ENOMEM) {
            JNU_ThrowOutOfMemoryError(env, "Map failed");
            return IOS_THROWN;
        }
        return handle(env, -1, "Map failed");
    }

    return ((jlong) (unsigned long) mapAddress);
}
```
这个方法主要是调用了`mmap64`系统调用，这个方法可以把它看作是`mmap`，它的作用是将文件的一个区间映射到进程的虚拟地址空间。


通过调用`map0`得到的映射区域的地址，保存在addr变量，然后，通过`Util.newMappedByteBuffer`方法来构造MappedByteBuffer的实例。



#### MappedByteBuffer的访问
看到`DirectByteBuffer`的get:
```
private long ix(int i) {
    return address + ((long)i << 0);
}

public byte get() {
    return ((unsafe.getByte(ix(nextGetIndex()))));
}

public byte get(int i) {
    return ((unsafe.getByte(ix(checkIndex(i)))));
}
```
可以看到，这里是直接访问内存地址的。


---
layout: post
title: "NIO的Selector"
date: 2018-08-15 22:36
comments: false
tags: 
- Java
- NIO
categories:	
- 基础
- Java
---

这篇文章介绍Selector的原理，并且通过它的一些native方法，分析Selector的几个重要操作如：创建、注册、选择。

<!--more-->

### ❄️ Selector的概述

#### 作用
SelectableChannel的多路复用器(multiplexor)
 
#### 流程
* 打开一个 Selector
* 将 Channel 注册到 Selector 中, 并设置需要监听的事件(interest set)
* 重复以下步骤：
  * 调用 select() 方法
  * 调用 selector.selectedKeys() 获取 selected keys
  * 迭代每个 selected key:
    * 从 selected key 中获取对应的 Channel 和附加信息(如果有的话)
    * 处理就绪的IO事件
    * 根据需要更改 selected key 的监听事件
    * 将已经处理过的 key 从 selected keys 集合中删除


#### SelectionKey
* SelectionKey代表一个 selectable channel 在1个 Selector上的注册。
* 一个Selector维护3个SelectionKey的集合：
  * <font color=green> **registered-key （已注册的键）**</font>：
  <font color=orange>与选择器关联的已经注册的键的集合</font> 。并不是所有注册过的键都仍然有效。这个集合通过keys( )方法返回，并且可能是空的。这个已注册的键的集合不是可以直接修改的；试图这么做的话将引java.lang.UnsupportedOperationException。 
  * <font color=green> **selected-key  （已选择的键）**</font>： 已注册的键的集合的子集。<font color=orange>这个集合的每个成员都是相关的通道被选择器（在前一个选择操作中）判断为已经准备好的，并且包含于键的interest集合中的操作。</font> 这个集合通过selectedKeys( )方法返回（并有可能是空的）。

    不要将已选择的键的集合与ready集合弄混了。这是一个键的集合，每个键都关联一个已经准备好至少一种操作的通道。每个键都有一个内嵌的ready集合，指示了所关联的通道已经准备好的操作。
    
    键可以直接从这个集合中移除，但不能添加。试图向已选择的键的集合中添加元素将抛出java.lang.UnsupportedOperationException。
  * <font color=green> **cancelled-key （已取消的键）**</font>：已注册的键的集合的子集，这个集合包含了cancel()方法被调用过的键（这个键已经被无效化），但它们还没有被注销。这个集合是选择器对象的私有成员，因而无法直接访问。
  
* 每一个SelectionKey包含2个以整数表示的“operation set”：
  * interest set：表示关心的操作
  * ready set ： 表示准备好的操作


#### Select操作
* 已取消的键的集合将会被检查。如果它是非空的，每个已取消的键的集合中的键将从另外两个集合中移除，并且相关的通道将被注销。这个步骤结束后，已取消的键的集合将是空的。 
* 已注册的键的集合中的键的interest集合将被检查。在这个步骤中的检查执行过后，对interest集合的改动不会影响剩余的检查过程。 

  一旦就绪条件被定下来，底层操作系统将会进行查询，以确定每个通道所关心的操作的真实就绪状态。依赖于特定的select()方法调用，如果没有通道已经准备好，线程可能会在这时阻塞，通常会有一个超时值。
  
  直到系统调用完成为止，这个过程可能会使得调用线程睡眠一段时间，然后当前每个通道的就绪状态将确定下来。对于那些还没准备好的通道将不会执行任何的操作。对于那些操作系统指示至少已经准备好interest集合中的一种操作的通道，将执行以下两种操作中的一种：
   * 如果通道的键还没有处于已选择的键的集合中，那么键的ready集合将被清空，然后表示操作系统发现的当前通道已经准备好的操作的比特掩码将被设置。 
   * 否则，也就是键在已选择的键的集合中。键的ready集合将被表示操作系统发现的当前已经准备好的操作的比特掩码更新。所有之前的已经不再是就绪状态的操作不会被清除。事实上，所有的比特位都不会被清理。由操作系统决定的ready集合是与之前的ready集合按位分离的，一旦键被放置于选择器的已选择的键的集合中，它的ready集合将是累积的。比特位只会被设置，不会被清理。
   
* 步骤2可能会花费很长时间，特别是所激发的线程处于休眠状态时。与该选择器相关的键可能会同时被取消。当步骤2结束时，步骤1将重新执行，以完成任意一个在选择进行的过程中，键已经被取消的通道的注销。
* select操作返回的值是ready集合在步骤2中被修改的键的数量，而不是已选择的键的集合中的通道的总数。返回值不是已准备好的通道的总数，而是从上一个select()调用之后进入就绪状态的通道的数量。之前的调用中就绪的，并且在本次调用中仍然就绪的通道不会被计入，而那些在前一次调用中已经就绪但已经不再处于就绪状态的通道也不会被计入。这些通道可能仍然在已选择的键的集合中，但不会被计入返回值中。返回值可能是0。 

### ❄️ Selector的代码分析

#### ⛄️ 创建选择器
我们通过调用`Selector::open`方法，创建选择器。而这个方法里面的逻辑如下：
```
// 1. Selector::open 通过使用SelectorProvider创建选择器，
//    这是因为这个是平台相关的
public static Selector open() throws IOException {
    return SelectorProvider.provider().openSelector();
}

// 2. 使用 DefaultSelectorProvider
public static SelectorProvider provider() {
    // ... 省略
    provider = sun.nio.ch.DefaultSelectorProvider.create();
}


// 3. 创建 SelectorProvider 的操作在这个方法里。（Linux环境）
//    如果Linux版本 >= 2.6 ，就使用 EPollSelectorProvider
//    如果Linux版本 <  2.6 ，就使用 PollSelectorProvider
public static SelectorProvider create() {
    // ... 省略
    
    // use EPollSelectorProvider for Linux kernels >= 2.6
    if ("Linux".equals(osname)) {
        String osversion = AccessController.doPrivileged(
            new GetPropertyAction("os.version"));
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


// 4.1 对于 PollSelectorProvider，它的openSelector()方法
public AbstractSelector openSelector() throws IOException {
    return new PollSelectorImpl(this);
}


// 4.2 对于 EPollSelectorProvider，它的openSelector()方法
public AbstractSelector openSelector() throws IOException {
    return new EPollSelectorImpl(this);
}
```


#### ⛄️ 将 Channel 注册到选择器
一般的用法是：
```
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```
##### AbstractSelectableChannel的register方法：
```
public final SelectionKey register(Selector sel, int ops,
                                   Object att)
    throws ClosedChannelException
{
    synchronized (regLock) {
        if (!isOpen())
            throw new ClosedChannelException();
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (blocking)
            throw new IllegalBlockingModeException();
        SelectionKey k = findKey(sel);
        if (k != null) {
            k.interestOps(ops);
            k.attach(att);
        }
        if (k == null) {
            // New registration
            synchronized (keyLock) {
                if (!isOpen())
                    throw new ClosedChannelException();
                k = ((AbstractSelector)sel).register(this, ops, att);
                addKey(k);
            }
        }
        return k;
    }
}
```
上面代码的逻辑如下：
1. 检查 channel是否关闭，设置的ops(channel的关心事件)是否合法
2. 如果当前 channel已经在给定的selector注册了，那么获取对应的SelectionKey, 设置ops
3. 如果当前 channel没有注册，就调用selector的register方法，然后在channel的SelectionKey[]数组添加新的SelectionKey


##### Selector的register方法
看到：
```
// 1. 在 SelectorImpl.java 的 register方法
protected final SelectionKey register(AbstractSelectableChannel ch,
                                          int ops,
                                          Object attachment)
{
    if (!(ch instanceof SelChImpl))
        throw new IllegalSelectorException();
    SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
    k.attach(attachment);
    synchronized (publicKeys) {
        implRegister(k);
    }
    k.interestOps(ops);             // 看第3点 “SelectionKeyImpl的interestOps方法”
    return k;
}


// 2. 在 AbstractPollSelectorImpl.java 的 implRegister方法
protected void implRegister(SelectionKeyImpl ski) {
    synchronized (closeLock) {
        if (closed)
            throw new ClosedSelectorException();
        // Check to see if the array is large enough
        if (channelArray.length == totalChannels) {
            // Make a larger array
            int newSize = pollWrapper.totalChannels * 2;
            SelectionKeyImpl temp[] = new SelectionKeyImpl[newSize];
            // Copy over
            for (int i=channelOffset; i<totalChannels; i++)
                temp[i] = channelArray[i];
            channelArray = temp;
            // Grow the NativeObject poll array
            pollWrapper.grow(newSize);
        }
        channelArray[totalChannels] = ski;
        ski.setIndex(totalChannels);
        pollWrapper.addEntry(ski.channel);
        totalChannels++;
        keys.add(ski);
    }
}
```

##### SelectionKeyImpl的interestOps方法

```
public SelectionKey interestOps(int ops) {
    ensureValid();                  // 表示这个 SelectionKey 还没有被 cancel
    return nioInterestOps(ops);     
}


SelectionKey nioInterestOps(int ops) {      // package-private
    if ((ops & ~channel().validOps()) != 0)
        throw new IllegalArgumentException();
    channel.translateAndSetInterestOps(ops, this);
    interestOps = ops;
    return this;
}
```
继续看 `channel.translateAndSetInterestOps()`，这方法是根据 channel 而不一样的。下面以 `SocketChannelImpl`的实现为例：
```
/**
 * Translates an interest operation set into a native poll event set
 */
public void translateAndSetInterestOps(int ops, SelectionKeyImpl sk) {
    int newOps = 0;
    if ((ops & SelectionKey.OP_READ) != 0)
        newOps |= PollArrayWrapper.POLLIN;
    if ((ops & SelectionKey.OP_WRITE) != 0)
        newOps |= PollArrayWrapper.POLLOUT;
    if ((ops & SelectionKey.OP_CONNECT) != 0)
        newOps |= PollArrayWrapper.POLLCONN;
    sk.selector.putEventOps(sk, newOps);
}
```
这个方法做了2件事：
* <font color=blue>将 SelectionKey的事件转换为 PollArrayWrapper定义的事件</font>，而像POLLIN、POLLOUT的值，和定义在 poll.h 文件的变量是一样的。
* 调用 selector 的 `putEventOps`方法，调用了`pollWrapper::putEventOps`方法，它就是设置`pollfd`数据结构的值



##### 小结
将 Channel 注册到选择器，做了2步：
* 创建了 SelectionKey（设置好 interestOps ），然后 channel和selector都保存好 SelectionKey。
* 通过 pollWrapper 设置“pollfd”


#### ⛄️ 通过 Selector 选择 Channel
使用如下方法:
```
int n = selector.select(); 
```
⚠️ 注意：
* 这个方法是阻塞的，当且仅当以下情况才会返回：
  * 至少1个channel就绪
  * 这个 selector 的 `wakeup` 方法被调用
  * 当前线程被中断
* 这个方法的返回值表示：SelectionKey的数量（它们的ready-operation set被更新）


##### pollSelectorImpl的doSelect
```
protected int doSelect(long timeout)
        throws IOException
{
    if (channelArray == null)
        throw new ClosedSelectorException();
    processDeregisterQueue();
    try {
        begin();
        pollWrapper.poll(totalChannels, 0, timeout);
    } finally {
        end();
    }
    processDeregisterQueue();
    int numKeysUpdated = updateSelectedKeys();
    if (pollWrapper.getReventOps(0) != 0) {
        // Clear the wakeup pipe
        pollWrapper.putReventOps(0, 0);
        synchronized (interruptLock) {
            IOUtil.drain(fd0);
            interruptTriggered = false;
        }
    }
    return numKeysUpdated;
}
```
上面代码做的是以下4件事：
* `processDeregisterQueue()`。
* 调用 pollWrapper的poll方法，它会调用 native 方法 `poll0`
* 调用`updateSelectedKeys`方法，从 pollfd 获取 revents 并转化、设置到对应的SelectedKey里面
* 检查这个 selector 的 wakeup 管道，如果有数据， 就把它“排干”，并且重置 pollfd 的 revents。


##### processDeregisterQueue
这个方法在 **SelectorImpl.*java*** ：
```
void processDeregisterQueue() throws IOException {
    // Precondition: Synchronized on this, keys, and selectedKeys
    Set cks = cancelledKeys();
    synchronized (cks) {
        if (!cks.isEmpty()) {
            Iterator i = cks.iterator();
            while (i.hasNext()) {
                SelectionKeyImpl ski = (SelectionKeyImpl)i.next();
                try {
                    implDereg(ski);
                } catch (SocketException se) {
                    IOException ioe = new IOException(
                        "Error deregistering key");
                    ioe.initCause(se);
                    throw ioe;
                } finally {
                    i.remove();
                }
            }
        }
    }
}
```
上面方法做的是：遍历 cancelledKeys集合，对于每一个元素（SelectionKeyImpl），使用方法 `implDereg`执行注销逻辑。这个方法在 ***AbstractPollSelectorImpl.java*** ：
```
// The list of SelectableChannels serviced by this Selector
protected SelectionKeyImpl[] channelArray;

// The number of valid channels in this Selector's poll array
protected int totalChannels;

protected void implDereg(SelectionKeyImpl ski) throws IOException {
    // Algorithm: Copy the sc from the end of the list and put it into
    // the location of the sc to be removed (since order doesn't
    // matter). Decrement the sc count. Update the index of the sc
    // that is moved.
    int i = ski.getIndex();
    assert (i >= 0);
    if (i != totalChannels - 1) {
        // Copy end one over it
        SelectionKeyImpl endChannel = channelArray[totalChannels-1];
        channelArray[i] = endChannel;
        endChannel.setIndex(i);
        pollWrapper.release(i);
        PollArrayWrapper.replaceEntry(pollWrapper, totalChannels - 1,
                                      pollWrapper, i);
    } else {
        pollWrapper.release(i);
    }
    // Destroy the last one
    channelArray[totalChannels-1] = null;
    totalChannels--;
    pollWrapper.totalChannels--;
    ski.setIndex(-1);
    // Remove the key from keys and selectedKeys
    keys.remove(ski);
    selectedKeys.remove(ski);
    deregister((AbstractSelectionKey)ski);
    SelectableChannel selch = ski.channel();
    if (!selch.isOpen() && !selch.isRegistered())
        ((SelChImpl)selch).kill();
}
```
上面方法的解析：
* 在这个 SelectorImpl 内部，保存了成员变量channelArray（SelectionKeyImpl[]）
* 这个方法执行的是 selector对SelectionKey的注销，包括删除channelArray中的元素、同步到 pollWrapper
* 还有是调用 deregister方法，它调用这个 SelectionKey对应的 Channel的 removeKey方法，也是删除 Channel保存的SelectionKey、keyCount 减1。
* 检查 Channel，如果已关闭并且没有注册到任何selector上（即keyCount == 0），那么使用 Channel 的 kill方法。

🍓 【联系】
 这里 Channel 的 kill方法，调用了 NativeDispatcher 的 close方法，<font color=pink>**为了避免多个线程下共享的fd的关闭释放而导致某些线程可能读写“旧”fd的问题，java nio的实现采用了two-step的关闭**</font>。



##### poll0的实现
```
JNIEXPORT jint JNICALL
Java_sun_nio_ch_PollArrayWrapper_poll0(JNIEnv *env, jobject this,
                                       jlong address, jint numfds,
                                       jlong timeout)
{
    struct pollfd *a;
    int err = 0;

    a = (struct pollfd *) jlong_to_ptr(address);

    if (timeout <= 0) {           /* Indefinite or no wait */
        RESTARTABLE (poll(a, numfds, timeout), err);
    } else {                     /* Bounded wait; bounded restarts */
        err = ipoll(a, numfds, timeout);
    }

    if (err < 0) {
        JNU_ThrowIOExceptionWithLastError(env, "Poll failed");
    }
    return (jint)err;
}
```
上面代码本质就是调用 poll 系统调用。根据 timeout的值，有2种情况：
* 如果 timeout <= 0， 就使用 宏定义 RESTARTABLE，死循环调用 poll，忽略它的EINTR
* 如果 timeout > 0，调用方法 ipoll，它与RESTARTABLE相比，多了超时计算。

##### updateSelectedKeys 的实现
```
/**
 * Copy the information in the pollfd structs into the opss
 * of the corresponding Channels. Add the ready keys to the
 * ready queue.
 */
protected int updateSelectedKeys() {
    int numKeysUpdated = 0;
    // Skip zeroth entry; it is for interrupts only
    for (int i=channelOffset; i<totalChannels; i++) {
        int rOps = pollWrapper.getReventOps(i);
        if (rOps != 0) {
            SelectionKeyImpl sk = channelArray[i];
            pollWrapper.putReventOps(i, 0);
            if (selectedKeys.contains(sk)) {
                if (sk.channel.translateAndSetReadyOps(rOps, sk)) {
                    numKeysUpdated++;
                }
            } else {
                sk.channel.translateAndSetReadyOps(rOps, sk);
                if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                    selectedKeys.add(sk);
                    numKeysUpdated++;
                }
            }
        }
    }
    return numKeysUpdated;
}
```
这个方法就是执行 poll 之后，从 pollfd 中取出 revents，设置到 SelectionKey，具体步骤如下：
* 通过 pollWrapper 取出 第 i 个 channel 的 revent。如果 revent == 0，处理下一个channel 的 revent
* 重置这个 pollfd 的 revents 为0， 取出对应的 SelectionKey 对象
* 调用 translateAndSetReadyOps方法，将对应的 已准备 ops 设置到 SelectionKey
* 如果这个 SelectionKey 不在 selectedKey 集合，还要添加 这个 SelectionKey到selectedKey 集合。


##### 小结
Select操作（以 pollSelectorImpl 为例，通过系统调用 poll 实现），做了以下几件事：
* 遍历处理 canceledKey-set
* 通过系统调用 poll，确定每个通道关心的操作的就绪状态。
* 将 poll 的结果同步到 SelectedKey。
* 返回 SelectedKey 被更新的数量
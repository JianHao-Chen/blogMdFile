---
layout: post
title: "Synchronized底层实现—-概论"
date: 2019-06-23 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

学习JVM源码Synchronized锁部分，写了如下几篇文章：
* Synchronized底层实现—-概论（本文）
* Synchronized底层实现—-偏向锁
* Synchronized底层实现—-轻量级锁
* Synchronized底层实现—-重量级锁

<!--more-->


## Java对象头
synchronized用的锁是存在Java对象头里的。关于对象头,有以下知识点:

☆ 对象在堆里的逻辑结构 :
  { 对象头 , 实例变量 }
  
☆ 如果对象是数组类型,虚拟机使用3个字宽存储对象头(用于存储数组长度),否则使用2个字宽。32位机下，1字宽  = 4字节

☆ 对象头的结构 :

  * Mark Word  (存储对象的hashCode 或锁信息)
  * Class Metadata Address (存储到对象类型数据的指针)
  * Array Length (数组的长度(如果当前对象是数组))
  
对象头的信息都是与对象自身定义的数据无关的额外存储成本,考虑到虚拟机的空间效率,Mark Word被设计成一个非固定的数据结构以便尽量存储多点信息,它会根据对象锁的状态复用自己的存储空间。

☆ Mark Word的结构

在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化。

<img src="/assets/blogImg/JVM/Synchronized/Introduce/1.png" width=600>

注意：
* Mark Word里有一个bit用于指示"是否是偏向锁",以此可以区分"未锁定"和"可偏向"
* Epoch字段：用于辅助批量重偏向（bulk rebiasing）和批量偏向撤销（bulk revocation）操作


## 锁的升级
<font color=red>**为了减少获得锁和释放锁的开销,引入"偏向锁"和"轻量级锁"。**</font>

Java1.6 中，锁一共有4种状态，级别从低到高分别是:
```
graph LR
无锁状态-->偏向锁状态
偏向锁状态-->轻量级锁状态
轻量级锁状态-->重量级锁状态
```

这几个状态会随着竞争情况逐渐升级,锁可以升级也可以降级。

### 偏向锁
**大多数情况,锁不但不存在多线程竞争,而且总是由同一个线程多次获得,为了降低获得锁的代价引入偏向锁**。

#### 初始状态
JVM默认是启用了偏向锁模式的。当新创建一个对象的时候，它的`mark word`将是可偏向状态，它的 thread id 为 0 ，表示未偏向任何线程，此时也叫做匿名偏向(anonymously biased)

#### 偏向锁的获取
1. 线程进入同步块，查看对象头Mark Word的锁标志位，如果是“01”，表示可偏向，进行偏向锁的获取。
2. 查看“是否是偏向锁”的bit位，如果是0，表示未锁定。进入步骤3。如果是1，表示当前已经是偏向锁，进入步骤4。
3. 执行CAS操作，替换 ThreadID。如果成功，表示已经获得偏向锁。如果失败，需要执行【偏向锁的撤销】。
4. 检查对象头的Mark Word中记录的是不是当前线程ID，如果是表示已经获得偏向锁。否则，进入步骤3。

#### 偏向锁的撤销
偏向锁使用一种等到竞争出现才释放锁的机制。偏向锁的撤销需要等待全局安全点(这个时候没有正在执行的字节码)。
1. JVM会挂起拥有偏向锁的线程,检查这个线程是否活着，如果这个线程不是活动状态或已经退出同步块，那么进入步骤2。否则进入步骤3。
2. 将对象头设置为无锁状态，并且唤醒原拥有偏向锁的线程。然后继续尝试获取偏向锁，即进入获取偏向锁的第4步。
3. 执行【升级为轻量级锁】的操作。

<font color=blue>补充 :</font>
偏向锁的释放不需要做任何事情,这也就意味着加过偏向锁的Mark Word会一直保留偏向锁的状态,因此即便同一个线程持续不断地加锁解锁，也是没有开销的。


#### 批量重偏向和批量偏向撤销
当偏向锁发生锁竞争时，就需要等到safe point时将偏向锁撤销为无锁状态或升级为轻量级/重量级锁，因此，**偏向锁的撤销是有一定成本的**。

因此，如果运行时的场景本身存在多线程竞争的，那偏向锁的存在不仅不能提高性能，而且会导致性能下降。因此，JVM中增加了一种批量重偏向/撤销的机制。

存在如下两种情况：
1. 一个线程创建了大量对象并执行了初始的同步操作，之后在另一个线程中将这些对象作为锁进行之后的操作。这种case下，会导致大量的偏向锁撤销操作。
2. 存在明显多线程竞争的场景下使用偏向锁是不合适的，例如生产者/消费者队列。


**批量重偏向（bulk rebias）** 机制是为了解决第一种场景，优化思路是<font color=green>**将前一个线程拥有的偏向锁重偏向到后一个线程**</font>。

**批量撤销（bulk revoke）** 则是为了解决第二种场景，优化思路是<font color=green>**禁用偏向锁! 将符合的对象的偏向锁撤销**</font>。

具体逻辑是：
* 以class为单位，为每个class维护一个偏向锁撤销计数器，每一次该class的对象发生偏向撤销操作时，该计数器+1，当这个值达到<font color=purple>**重偏向阈值（默认20）**</font>时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向（rebias）。
* 对所有属于这个类的对象进行重偏向的操作叫**批量重偏向（bulk rebias）**。
* 每个class对象会有一个对应的epoch字段，每个处于偏向锁状态对象的mark word中也有该字段，其初始值为创建该对象时，class中的epoch的值。
* 每次发生批量重偏向时，就将class的epoch值+1，同时遍历JVM中所有线程的栈，找到该class所有正处于加锁状态的偏向锁，将其epoch字段改为新值。
* 下次获取锁时，发现当前对象的epoch值和class的epoch不相等，那就算当前已经偏向了其他线程，也不会执行撤销操作，而是直接通过CAS操作将其mark word的Thread Id 改成当前线程Id。
* 当达到重偏向阈值后，假设该class计数器继续增长，当其达到<font color=purple>**批量撤销的阈值（默认40）**</font>后，JVM就认为该class的使用场景存在多线程竞争，会标记该class为不可偏向，之后，对于该class的锁，直接走轻量级锁的逻辑。


<font color=red>注意 </font>：
* 判断一个对象是否获得偏向锁需要同时满足以下条件：
  * mark字段后3位是“101”
  * 锁对象的`thread`字段跟当前线程相同
  * 锁对象的`epoch`字段跟所属类的epoch值相同
* 批量重偏向的操作，仍然需要在安全点执行


#### 获取偏向锁的流程
如下面的伪代码所示（引自论文 Eliminating Synchronization-RelatedAtomic Operations with Biased Locking and Bulk Rebiasing）：
```
void lock (Object* obj, Thread* t) {
    int lw = obj->lock_word;
    if (lock_state(lw) == Biased && biasable(lw) == obj->class->biasable
       && bias_epoch(lw) == obj->class->epoch) {
       if (lock_or_bias_owner == t->id) {
           // current thread is the bias owner
           return;
       } else {
           // need to revoke the object bias
           revoke_bias(obj, t);
       }
    } else {
       // normal locking/unlocking protocal,
       // possibly with bias acquisition.
    {
}
```





### 轻量级锁
**引入的目的是在没有多线程竞争的前提下，减少传统的重量级锁使用产生的性能消耗**。

因为有着一个经验数据：“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”。没有竞争，轻量级锁使用CAS操作避免使用互斥量的开销。但如果存在锁竞争，除了互斥量的开销以为，还额外发生了CAS操作，因此在有竞争时轻量级锁会比传统重量级锁更慢。

#### 轻量级锁的获取

☆ 情况1 (线程进入同步块)
1. 线程进入同步块，查看对象头Mark Word的锁标志位，如果是“00”，表示当前已经是轻量级锁，进行轻量级锁的获取。
2. 首先在当前线程的栈帧中创建用于存储锁记录的空间，并拷贝对象头中的Mark Word到当前线程的锁记录中(官方称之为 displaced mark word)。
3. 通过CAS操作将对象头的Mark Word替换成指向锁记录的指针。如果成功，即获得轻量级锁。否则，进入步骤4。
4. 通过自旋，重复步骤3，失败一定次数后，执行【升级为重量级锁】操作。

☆ 情况2 (获取偏向锁失败升级为轻量级锁)
1. 在原持有偏向锁的线程的栈帧中创建用于存储锁记录的空间，并拷贝对象头中的Mark Word到这个线程的锁记录中，这样这个线程就获得轻量级锁。
2. 原持有偏向锁的线程被JVM唤醒，在安全点处继续执行。当前线程则从情况1的步骤2开始执行。


#### 轻量级锁的解锁
1. JVM使用CAS操作--如果对象的Mark Word仍然指向着线程的锁记录，那就用CAS操作把对象当前的Mark Word和线程中复制的Displaced Mark Word替换回来。
2. 如果替换成功，整个同步过程就完成了。
3. 如果替换失败，表示有其他线程尝试过获取该锁，存在锁竞争，锁已经膨胀为重量级锁，而且其他的线程已经阻塞于这个锁，当前线程在释放锁之后会唤醒这些线程。这些线程会开始新一轮的锁竞争。


#### synchronized的实现原理图

<img src="/assets/blogImg/JVM/Synchronized/Introduce/3.png" width=700>

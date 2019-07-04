---
layout: post
title: "HotSpot内存管理基础知识"
date: 2019-06-15 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文主要回顾Java GC方面的知识。

<!--more-->

### 垃圾收集器
负责释放不再被使用的对象所占空间的程序模块，叫垃圾收集器（Garbage Collector）。

#### 主要任务
* 确保仍被引用的对象在内存中保持存在
* 回收无任何引用的对象所占用的空间。

同时，垃圾收集器还应该做到：
* 高效：不应该导致应用出现长时间的停顿。
* 减少内存碎片：一个消除内存碎片的方法--压缩（compaction）。


#### 相关策略
* GC工作线程：串行还是并行？
* GC工作线程与应用线程： 并发还是暂停应用？
* 基本收集算法：压缩？非压缩？拷贝？


#### 垃圾收集器的性能指标
* 吞吐量：应用程序运行时间/(应用程序运行时间 + 垃圾收集时间)。即没有花在垃圾收集的时间占总时间的比例。
* 垃圾收集开销：表示垃圾收集耗用时间占总时间的比例。
* 暂停时间：垃圾收集过程中，应用程序被停止的时间。
* 收集频率：垃圾收集的频率
* 堆空间：堆空间所占内存大小。


### 收集算法
GC常用的基本收集算法有3种：

#### 标记-清除（Mark-Sweep）
这个算法分为2个阶段：
* 标记阶段：标记出所有可以回收的对象
* 清除阶段：回收所有已标记的对象


##### 复制算法 (Copping)
* 划分区域：将内存区域按比例划分为1个Eden区作为分配对象的“主战场”和2个幸存区（Survivor空间，划分为2个等比例的from、to区）。
* 复制：收集时，打扫“战场”，将Eden区中仍存活的对象复制到某一块幸存区中。
* 清除：由于上一阶段已确保仍存活对象已被妥善安置，现在可以“清理战场”了，释放Eden区和另一块幸存区。
* 晋升：若在“复制”阶段，一块幸存区接纳不了所有的“幸存”对象，就直接晋升到老年代。

#####  标记-压缩算法 (Mark-Compact)
此算法分为2个阶段：
* 标记阶段：标记出所有可以回收的对象
* 压缩阶段：将标记阶段的对象移动到空间的一端，释放剩余空间。


一般来说，==复制算法适合新生代的收集，基于标记的算法则适合老年代的收集==。


###  分代收集
由于对象生命周期有很大差别，所以采用不同收集策略，因此分代收集就产生了。

#### 分代收集
分代收集（Generational Collection）是指在内存空间中划分不同的区域，在各自区域中分别存储不同年龄的对象，每个区域可以根据自身的特点灵活采用自身的收集策略。有以下3种类型：
* 新生代（Young Generation）
  *  划分为1个Eden区和2个幸存区（1个from和1个to区）
* 老年代（Old Generation）
* 永久代（Permanent Generation），Java8移除永久代。

#### 垃圾收集类型
根据垃圾收集作用在不同分代，垃圾收集类型分为2种：
* Minor Collection：对新生代进行收集
* Full Collection：对所有的分代都进行收集


### HotSpot GC收集器

<img src="/assets/blogImg/JVM/GC/Introduce/1.png" width=400>

#### 各个收集器的简介
* Serial 收集器：
  * 作用于新生代，基于复制算法
  * 单线程
* Serial Old 收集器：
  * 作用于老年代，基于“标记-整理” (mark-sweep-compact)算法
  * 单线程
* ParNew收集器：
  * 作用于新生代，基于复制算法
  * 多线程
* Parallel Scavenge收集器：
  * 作用于新生代，基于复制算法
  * 多线程
* Parallel Old收集器：
  * 作用于老年代，基于“标记-整理” (mark-sweep-compact)算法
  * 多线程
* CMS收集器：
  * 作用于老年代，基于“标记-清除” (mark-sweep)算法
  * 多线程


#### VM参数与收集器

VM参数 | 使用的收集器 | 备注
---|--- | --
UseSerialGC | Serial + Serial Old
UseParNewGC | ParNew + Serial Old
UseConcMarkSweepGC | ParNew + CMS + Serial Old | CMS用于老年代，Serial Old在CMS收集过程中发生concurrent mode failure时使用
UseParallelGC | Parallel Scavenge + Serial Old |
UseParallelOldGC | Parallel Scavenge + Parallel Old |

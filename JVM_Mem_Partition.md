---
layout: post
title: "HotSpot内存职能划分"
date: 2019-06-13 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---



#### HotSpot 内存职能划分

<img src="/assets/blogImg/JVM/Mem_Partition/1.png" width=400>


#### 总结：
* 在 HotSpot实现，Java栈和本地方法栈是合二为一的，<font color=orange>在本地内存空间中分配</font>。
* 堆和方法区由所有线程共享，由JVM的内存管理模块维护。
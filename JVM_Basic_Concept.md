---
layout: post
title: "JVM和JRE"
date: 2019-06-01 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

#### JDK 和 JRE 的区别

![](/assets/blogImg/JVM/Basic_Concept/1.png)

<!--more-->

#### JVM 的组成

![](/assets/blogImg/JVM/Basic_Concept/2.png)

* JVM由两个主要组件构成：执行引擎和运行时。
* JVM 和 Java API 组成 Java 运行环境，也称为 JRE。


##### 执行引擎
执行引擎由两个主要组件构成：
* 垃圾回收器：不解释
* 即时编译器：把字节码转换为可执行的机器码


##### 运行时
负责以下功能：
* VM lifecycle
* 类加载
* 解析（Interpreter）
* 线程管理
* 。。。







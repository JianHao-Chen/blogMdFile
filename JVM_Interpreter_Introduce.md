---
layout: post
title: "HotSpot解释器介绍"
date: 2019-06-22 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---



### 编译和解释
高级语言需要经过“翻译”，才能让机器执行。“翻译”有2种方式：
* 编译：由编译器将高级语言编译成机器可识别的可执行文件如exe，直接执行可执行文件以完成程序的运行。
* 解释：（在运行时）由解释器“边翻译边执行”完成程序的运行。

Java是解释性语言，在HotSpot的运行时，除了有解释器，还有即时编译器。

<!--more-->


### 解释器和即时编译器

<img src="/assets/blogImg/JVM/Interpreter/1.png" width=500>

#### 解释器
翻译执行字节码，执行方式是一边翻译一边执行（效率低，但易实现）。

HotSpot中解释器由以下部分组成：
* 解释器（interpreter）: 提供解释执行的功能。HotSpot中有2种解释器：
  * 模版解释器（TemplateInterpreter，默认）
  * c++解释器（CppInterpreter）
* 代码生成器（code generator）：利用解释器的宏汇编器向代码缓存空间写入生成的代码
* InterpreterCodeLet：由解释器运行的代码片段。所有由代码生成器生成的代码都由一个CodeLet表示。面向解释器的CodeLet称为 InterpreterCodeLet，由解释器维护，利用这些CodeLet，JVM可以在内部存储、定位、执行代码任务。
* 转发表（dispatch table）：为方便快速找出字节码对应的机器码，模版解释器使用了转发表。

#### JIT编译器
HotSpot除了提供解释器，还可以对程序执行频繁的代码优化，将其编译为本地代码。这些频繁执行的代码称为“热点”代码（HotSpot因此得名）。

执行编译任务的组件叫JIT编译器。
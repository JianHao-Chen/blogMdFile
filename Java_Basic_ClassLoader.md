---
layout: post
title: "ClassLoader的原理和API"
date: 2018-5-07 10:12
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

这篇文章主要是整理学习过程中Java的ClassLoader的一些知识点。

<!--more-->

### ClassLoader的原理

#### 职责
1. 根据一个指定的类的名称,找到或者生成其对应的字节代码，然后从这些字节代码中定义出一个 Java 类，即 java.lang.Class类的一个实例。
2. ClassLoader还负责加载 Java 应用所需的资源，如图像文件和配置文件等。

【注意】
 * 每一个Class对象,都有一个指向它的ClassLoader的引用。
 * 数组类型的Class对象,不是由ClassLoader生成的，而是由JVM创建的。


#### ClassLoader的树状组织结构
Java 中的类加载器大致可以分成两类，一类是系统提供的，另外一类则是由 Java 应用开发人员编写的。

系统提供的类加载器有下面三个:
1. 引导类加载器(bootstrap class loader)
它用来加载 Java 的核心库，是用C++来实现的，在Java中看不到它，是null。它用来加载核心类库，就是在lib下的类库。
2. 扩展类加载器(extensions class loader)
加载lib/ext下的类库
3. 系统类加载器(system class loader)
它根据 Java 应用的类路径（CLASSPATH）来加载 Java 类。一般来说，Java 应用的类都是由它来完成加载的。可通过 ClassLoader.getSystemClassLoader()来获取它。

开发人员可以通过继承 java.lang.ClassLoader类的方式实现自己的类加载器.


#### Java 虚拟机是如何判定两个 Java 类是相同的
Java 虚拟机不仅要看类的全名是否相同，还要看加载此类的类加载器是否一样。只有两者都相同的情况，才认为两个类是相同的。即便是同样的字节代码，被不同的类加载器加载之后所得到的类，也是不同的。


#### ClassLoader的委托模型(delegation model)
类加载器在尝试自己去查找某个类的字节代码并定义它时，会先代理给其父类加载器，由父类加载器先去尝试加载这个类。于是,加载一个类时，首先BootStrap进行寻找，找不到再由Extension ClassLoader寻找，最后才是App ClassLoader。


#### defining loader 和 initiating loader
加载类的时候，类加载器会首先代理给其它类加载器来尝试加载某个类,这就意味着真正完成类的加载工作的类加载器和启动这个加载过程的类加载器，有可能不是同一个。

☆ 真正完成类的加载工作是通过调用 defineClass来实现的。
☆ 而启动类的加载过程是通过调用 loadClass来实现的

前者称为一个类的定义加载器（defining loader）,后者称为初始加载器（initiating loader）。

JVM判断两个类是否相同的时候，使用的是类的定义加载器。


#### 类实例缓存
类加载器在成功加载某个类之后，会把得到的 java.lang.Class类的实例缓存起来。下次再请求加载该类的时候，类加载器会直接使用缓存的类的实例，而不会尝试再次加载。也就是说，对于一个类加载器实例来说，相同全名的类只加载一次，即 loadClass方法不会被重复调用。

在JVM中，类型被定义在一个叫 SystemDictionary 的数据结构中，该数据结构接受类加载器和全类名作为参数，返回类型实例。类型加载时，需要传入类加载器和需要加载的全类名，如果在 SystemDictionary 中能够命中一条记录，则返回class 列上对应的类型实例引用，如果无法命中记录，则会调用loader.loadClass(name);进行类型加载。


### ClassLoader的几个关键的方法

* findLoadedClass()方法
 ```
 // 根据给定的名字,查找被此ClassLoader加载的类(作为initiating loader)。
 protected final Class<?> findLoadedClass(String name)
 ```

* findClass()方法
 ```
 // 查找名称为 name的类。
 protected Class<?> findClass(String name) throws ClassNotFoundException
 ```
 这个方法默认是直接抛出ClassNotFoundException。所以应该由子类override这个方法。
 当loadClass(String name)方法调用，在当前ClassLoader的缓存没有找到，并且其父类加载器也没有找到时，会调用此方法。

* defineClass()方法
 这个方法被声明为 final的 ！！
 ```
 protected final Class<?> defineClass(String name, byte[] b, int off, int len)
 ```
 这个方法接受一组字节，然后将其具体化为一个Class类型实例，它将文字节传递给JVM，通过JVM（native 方法）对于Class的定义，将其具体化，实例化为一个Class类型实例。
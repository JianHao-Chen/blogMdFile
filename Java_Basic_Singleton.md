---
layout: post
title: "Java单例模式的7种写法"
date: 2017-11-03 14:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

介绍Java单例模式的7种写法。

<!--more-->

单例模式要求类能够有返回对象一个引用(永远是同一个)和一个获得该实例的方法（必须是静态方法）。

单例的实现有以下两个步骤：
1. 将该类的构造方法定义为私有方法，这样其他的代码就无法通过调用该类的构造方法来实例化该类的对象，只有通过该类提供的静态方法来得到该类的唯一实例。
2. 在该类内提供一个静态方法，当我们调用这个方法时，如果类持有的引用不为空就返回这个引用，如果类保持的引用为空就创建该类的实例并将实例的引用赋予该类保持的引用。

### 单例模式的7种写法

#### 饿汉式(静态常量)【可用】

```bash
public class Singleton {
  private final static Singleton INSTANCE = new Singleton();
  private Singleton(){}
  public static Singleton getInstance(){
    return INSTANCE;
  }
}
```
优点：这种写法比较简单，就是在类装载的时候就完成实例化。避免了线程同步问题。
缺点：在类装载的时候就完成实例化，没有达到Lazy Loading的效果。如果从始至终从未使用过这个实例，则会造成内存的浪费。


#### 懒汉式(线程不安全)【不可用】
```bash
public class Singleton {
  private static Singleton singleton;
  private Singleton() {}

  public static Singleton getInstance() {
    if (singleton == null) {
      singleton = new Singleton();
    }
    return singleton;
  }
}
```
这种写法起到了Lazy Loading的效果，但是只能在单线程下使用。

#### 懒汉式(线程安全，同步方法)【不推荐】
```bash
public class Singleton {
  private static Singleton singleton;
  private Singleton() {}
  
  public static synchronized Singleton getInstance() {
    if (singleton == null) {
      singleton = new Singleton();
    }
    return singleton;
  }
}
```
缺点：效率太低了，每个线程在想获得类的实例时候，执行getInstance()方法都要进行同步。而其实这个方法只执行一次实例化代码就够了，后面的想获得该类实例，直接return就行了。

#### 懒汉式(线程不安全，同步方法)【不可用】
```bash
public class Singleton {
  private static Singleton singleton;
  private Singleton() {}

  public static Singleton getInstance() {
    if (singleton == null) {
      synchronized (Singleton.class) {
        singleton = new Singleton();
      }
    }
    return singleton;
  }
}
```
尝试将第3种写法改进，首先判断singleton引用是否为null，如果不是直接返回实例的引用，否则进入同步块创建实例。
这种写法是线程不安全的。假设：
线程A进入getInstance()方法，判断singleton==null，此时发生调度，线程B执行，进入getInstance()方法，同样判断singleton==null，于是进入同步块，创建实例并返回。当线程A继续执行时，也会进入同步块，创建实例。这样实例就不是唯一的。


#### 懒汉式(线程不安全，双重检查锁定)【不可用】
```bash
public class Singleton {
  private static Singleton singleton;
  private Singleton() {}

  public static Singleton getInstance() {
    if (singleton == null) {            //第一次检查
      synchronized (Singleton.class) {  // 加锁
        if(singleton == null)           //第二次检查
          singleton = new Singleton();  // ！！ 危险 ！！
      }
    }
    return singleton;
  }
}
```
这种写法是写法4的改进，似乎解决了问题，因为后进入同步块的线程是不可能看到`singleton == null`，不会重复生成实例。
但是，还有一个问题：<font color=red>当某个线程第一次检查`singleton == null`时，即使 singleton != null，singleton引用的对象可能还没有完成初始化！</font>

语句 `singleton = new Singleton();` 创建一个对象，其实可以分解为3步：
```bash
memory = allocate;    // 1. 分配对象的内存空间
Init(memory);         // 2. 初始化对象
singleton = memory;   // 3. 设置singleton指向memory
```
而第2步和第3步，可能被重排序。因此，可能出现以下这种时序：

| 时间         | 线程A           | 线程B  |
| ------------- |:-------------:|:-----:|
|t1| A1: 分配对象的内存空间 |  |
|t2| A3: 设置singleton指向内存空间     |    |
|t3||    B1: 判断singleton是否为空 |
|t4||    B2: 由于singleton不为null，线程B将访问singleton引用的对象 |
| t5   |  A2: 初始化对象  ||
| t6   |  A4: 访问singleton引用的对象      |<font color=white>;</font>|

既然知道了问题的根源，我们可以想出2种方法来解决：
* 不允许2和3重排序。
* 允许2和3重排序，但不允许其它线程“看到”这个重排序。


#### 懒汉式(线程安全，双重检查锁定+volatile)【可用】
```bash
public class Singleton {
  private volatile static Singleton singleton;
  private Singleton() {}

  public static Singleton getInstance() {
    if (singleton == null) {
      synchronized (Singleton.class) {
        if(singleton == null)
          singleton = new Singleton();
      }
    }
    return singleton;
  }
}
```
这种写法与写法5相比，只是将singleton声明为volatile。
这样，创建对象singleton时，步骤2和3的重排序会被禁止。

【注意】
这种解决方法需要JDK5或更高版本，因为从JDK5开始使用新的JSR-133内存模型规范，这个规范增强了volatile的语义。


#### 静态内部类(线程安全)
JVM在执行类的初始化(即在Class被加载后，且被线程使用之前)时，会获取一个锁，这个锁可以同步多个线程对同一个类的初始化。
因此，我们可以利用这个特性实现以下的单例模式：
```bash
public class Singleton {
  private Singleton() {}

  private static class SingletonInstance{
    private static final Singleton INSTANCE = new Singleton();
  }

  public static Singleton getInstance(){
    return SingletonInstance.INSTANCE;  //这里将导致SingletonInstance类被初始化
  }
}
```
这种方式采用了**不允许其它线程看到重排序**的策略，因为通过锁将其他线程排除在外了。

这种方式跟饿汉式方式采用的机制类似，但又有不同。两者都是采用了类装载的机制来保证初始化实例时只有一个线程。不同的地方在饿汉式方式是只要Singleton类被装载就会实例化，没有Lazy-Loading的作用，而静态内部类方式在Singleton类被装载时并不会立即实例化，而是在需要实例化时，调用getInstance方法，才会装载SingletonInstance类，从而完成Singleton的实例化。



#### 枚举【可用】
```bash
public enum Singleton {
  INSTANCE;
  public void whateverMethod() {
  }
}
```
借助JDK1.5中添加的枚举来实现单例模式。不仅能避免多线程同步问题，而且还能防止反序列化重新创建新的对象。可能是因为枚举在JDK1.5中才添加，所以在实际项目开发中，很少见人这么写过。
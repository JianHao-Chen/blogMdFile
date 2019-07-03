---
layout: post
title: "Java的final关键字"
date: 2018-5-05 10:12
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

这篇文章主要是整理学习过程中Java的final关键字的一些知识点。

<!--more-->

### final关键字的基本用法
在Java中，final关键字可以用来修饰类、方法和变量（包括成员变量和局部变量）。
下面就从这三个方面来了解一下final关键字的基本用法。

#### 修饰类
当用final修饰一个类时，表明这个类不能被继承。也就是说，如果一个类你永远不会让他被继承，就可以用final进行修饰。final类中的成员变量可以根据需要设为final，但是要注意final类中的所有成员方法都会被隐式地指定为final方法

#### 修饰方法
使用final方法的原因有两个:
1. 把方法锁定，以防任何继承类修改它的含义；
2. 效率。在早期的Java实现版本中，会将final方法转为内嵌调用。但是如果方法过于庞大，可能看不到内嵌调用带来的任何性能提升。在最近的Java版本中，不需要使用final方法进行这些优化了。

因此，如果只有在想明确禁止该方法在子类中被覆盖的情况下才将方法设置为final的。


#### 修饰变量
修饰变量是final用得最多的地方，也是本文接下来要重点阐述的内容。

首先了解一下final变量的基本语法：<font color=red>对于一个final变量，如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；如果是引用类型的变量，则在对其初始化之后便不能再让其指向另一个对象。</font> 
举个例子：
```
class A{
   final int i = 0;
   final Object o = new Object();
   
   void f(){
       i = 1;                // 赋值报错:The final field cannot be assigned
       o = new Object();     // 赋值报错:The final field cannot be assigned
   }
}
```

还有以下2点需要注意：
* 当final变量是基本数据类型以及String类型时，如果在编译期间能知道它的确切值，则编译器会把它当做编译期常量使用。也就是说在用到该final变量的地方，相当于直接访问的这个常量，不需要在运行时确定。
```
String a = "hello2"; 
final String b = "hello";
String d = "hello";
String c = b + 2; 
String e = d + 2;
System.out.println((a == c));      // true
System.out.println((a == e));      // false
```
  由于变量b被final修饰，因此会被当做编译器常量，所以在使用到b的地方会直接将变量b替换为它的值。
* 被final修饰的引用变量指向的对象内容是可变的。
```
class A{
   int i = 0;
}

public class Test {
    public static void main(String[] args){
        final A a = new A();
        a.i = 1;
    }
}
```


### final域的内存语义
从JDK 5开始，Java使用新的JSR-133内存模型，它增强了final的语义，解决了final域的重排序问题。

先看如下代码：
```
public class FinalExample {
    int i;
    final int j;
    static FinalExample  obj;
    
    public FinalExample(){
        i = 1;
        j = 2;
    }
    
    public static void writer(){
        obj = new FinalExample();
    }
    
    public static void reader(){
        FinalExample object = obj;
        int a = object.i;
        int b = object.j;
    }
    
}
```
假设一个线程A执行writer()方法，随后另一个线程B执行reader()方法。

#### 写final域的重排序规则
写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则的实现包含2个方面：
1. JMM禁止编译器把final域的写重排序到构造函数之外。
2. 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。

写final域的重排序规则可以确保：<font color=red>在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。</font>


#### 读final域的重排序规则
读final域的重排序规则是，在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这2个操作(这个规则仅仅针对处理器)。编译器会在读final域操作的前面插入一个LoadLoad屏障。

读final域的重排序规则可以确保：<font color=red>在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。</font>

---
layout: post
title: "局部内部类"
date: 2018-5-04 21:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

局部内部类就像是方法里面的一个局部变量一样，是不能有public、protected、private以及static修饰符的。局部内部类(也叫方法内部类)，定义在一个方法或者一个作用域里面的类，它和成员内部类的区别在于局部内部类的访问仅限于方法内或者该作用域内。

<!--more-->

注意:
方法内部类只能够访问该方法中的局部变量，而且这些局部变量一定要是final修饰的常量。

先看以下示例代码:
```
class Outter{
　　public void outMethod(){
　　　　final int beep = 99;
　　　　
　　　　//局部内部类
　　　　class Inner{
　　　　　　public void F(){
　　　　　　  //　这里访问Outter的变量b,如果去掉b的final修饰符，将会编译报错。
　　　　　　  System.out.println(beep);
　　　　　　}
　　　　｝
　　｝
｝
```
通过查看Outter的字节码，可以看到它只有构造器和outMethod()方法，没有返回私有域的隐藏方法了，确实这里Inner访问的并不是Outter的成员变量，而是outMethod方法内部的局部变量。

再看看Inner是如何访问outMethod()方法的局部变量beep的：
```
public void F();
  descriptor: ()V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=1, args_size=1
      0: getstatic     #20                 // Field java/lang/System.out:Ljava/io/PrintStream;
      3: bipush        99
      5: invokevirtual #26                 // Method java/io/PrintStream.println:(I)V
      8: return
    LineNumberTable:
      line 38: 0
      line 39: 8
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       9     0  this   Ljvm/innerClass/Outter$1Inner;
```
可以看到，对于F()方法中的“System.out.println(beep)”，对应为字节码:
```
3: bipush        99
5: invokevirtual #26                 // Method java/io/PrintStream.println:(I)V
```
表示先将99压到栈顶，然后调用println()方法。

由于编译器要求我们必须将beep定义为一个final变量，然后编译器就直接用常量值99了。
这个也难怪，因为当Inner的F()方法运行时，很有可能outMethod()已经运行完，局部变量beep也已经被回收了。F()需要访问beep，只能在Inner里面“复制”一份beep。

而如果beep不是一个final变量，那么一旦outMethod()中的beep改变了，在Inner中的那个副本beep也要跟着改变，为了保持局部变量与局部内部类中备份域保持一致，编译器只能规定这些局部域必须是常量。

既然是常量，编译器也就直接操作99这个数字常量，而没有在Inner中创建一个成员变量beep。
---
layout: post
title: "匿名内部类"
date: 2018-5-03 21:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

回答这个问题:
```bash
class Outter{
  public void go(){
    final int filed = 12;
    
    Thread t = new Thread("TestThread"){
      public void run(){
        System.out.println(filed);
      }
    };
  }
  
  public void goWithParameter(final int field){
  
    Thread t = new Thread(""){
      public void run(){
        System.out.println(field);
      }
    };
  }
}
```
不论是`go()`还是`goWithParameter()`，它们所访问的`field`都必须带有`final`修饰符，否则编译报错。为什么？

<!--more-->

先说答案：<font color=green>其实上面这种就叫匿名内部类，它只能访问局部final变量！！</font>

下面通过分析它们的字节码来搞明白其中的原理:

#### go()方法中的匿名内部类
编译器把这个内部类命名为"Outter$1"。它的run()方法的字节码如下 ：

```
public void run();
  descriptor: ()V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=1, args_size=1
      0: getstatic     #23                 // Field java/lang/System.out:Ljava/io/PrintStream;
      3: bipush        12
      5: invokevirtual #29                 // Method java/io/PrintStream.println:(I)V
      8: return
    LineNumberTable:
      line 68: 0
      line 69: 8
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
        0        9      0   this    Ljvm/innerClass/Outter$1;
```
在run方法中有一条指令：
```
bipush        12
```
这条指令表示将操作数12压栈，表示使用的是一个本地局部变量。这个过程是在编译期间由编译器默认进行，如果这个变量的值在编译期间可以确定，则编译器默认会在匿名内部类（局部内部类）的常量池中添加一个内容相等的字面量或直接将相应的字节码嵌入到执行字节码中。这样一来，匿名内部类使用的变量是另一个局部变量，只不过值和方法中局部变量的值相等，因此和方法中的局部变量完全独立开。

<font color=red>**为什么匿名内部类里面操作的是另一份变量？**</font>
当go方法执行完毕之后，变量filed的生命周期就结束了,而此时Thread对象的生命周期很可能还没有结束，那么在Thread的run方法中继续访问变量a就变成不可能了，但是又要实现这样的效果，怎么办呢？Java采用了 <font color=blue>**复制**</font> 的手段来解决这个问题。
为解决数据不一致性问题，java规定了将参数限制为final。


#### goWithParameter()方法中的匿名内部类
编译器把这个内部类命名为"Outter$2"。先看它的构造方法的字节码 ：
```
jvm.innerClass.Outter$2(jvm.innerClass.Outter, java.lang.String, int);
  descriptor: (Ljvm/innerClass/Outter;Ljava/lang/String;I)V
  flags:
  Code:
    stack=2, locals=4, args_size=4
      0: aload_0
      1: aload_1
      2: putfield      #12      // Field this$0:Ljvm/innerClass/Outter;
      5: aload_0
      6: iload_3
      7: putfield      #14      // Field val$field:I
      10: aload_0
      11: aload_2
      12: invokespecial #16     // Method java/lang/Thread."<init>":(Ljava/lang/String;)V
      15: return
    LineNumberTable:
      line 1: 0
      line 75: 10
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
        0      16     0    this   Ljvm/innerClass/Outter$2;
        0      16     2    $anonymous0   Ljava/lang/String;
```
这里是将变量goWithParameter方法中的形参field以参数的形式传进来对匿名内部类中的拷贝（变量field的拷贝）进行赋值初始化。

然后再看它的run()方法：
```
public void run();
  descriptor: ()V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=1, args_size=1
      0: getstatic     #27                 // Field java/lang/System.out:Ljava/io/PrintStream;
      3: aload_0
      4: getfield      #14                 // Field val$field:I
      7: invokevirtual #33                 // Method java/io/PrintStream.println:(I)V
      10: return
    LineNumberTable:
      line 77: 0
      line 78: 10
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0      11     0  this   Ljvm/innerClass/Outter_A$2;
```
与Outter$1的run()方法相比，它的run()方法在调用“System.out.println(field)”之前，调用了“getfield      #14”，这个就是在构造函数里面设置进去的goWithParameter()方法的入参值。
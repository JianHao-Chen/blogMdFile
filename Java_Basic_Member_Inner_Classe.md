---
layout: post
title: "成员内部类"
date: 2018-5-04 22:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

以下2个问题，你可以回答出来吗：
1. 为什么成员内部类可以无条件访问外部类的成员？
2. 如果内部类只有一个private的构造函数，外部类还可以创建内部类的对象吗？

<!--more-->

先看以下示例代码:

```
class Outter {
  private Inner inner = null;
  private int i = 0;
  private int c = 1;
  
  public Outter() {}
  
  /*
   *  在外部类中如果要访问成员内部类的成员，必须先创建一个成员内部类的对象，
   *  再通过指向这个对象的引用来访问
   */
  public void accessInnerClass(){
    getInnerInstance().print();
  }
  
  public Inner getInnerInstance() {
    if(inner == null)
      inner = new Inner();
    return inner;
  }
  
  class Inner {
    private Inner(){}
    private void print() {
      System.out.println(i);  //外部类的private成员
    }
    
    /*
     * 成员内部类不能有static 的field 和 method.
     * 原因 :
     *    类加载顺序是先static然后instance的。
    */
//    private static int y = 0;
//    private static void staticPrint() {}
  }
}
```

#### 创建成员内部类对象
成员内部类是依附外部类而存在的，也就是说，如果要创建成员内部类的对象，前提是必须存在一个外部类的对象。有以下2种写法，虽然本质是一样的:

```bash
//第一种方式：
Outter outter = new Outter();
Outter.Inner inner = outter.new Inner();  //必须通过Outter对象来创建

//第二种方式：
Outter outter = new Outter();
Outter.Inner inner1 = outter.getInnerInstance();
```

#### 为什么成员内部类可以无条件访问外部类的成员？
我们通过java的反编译命令 javap 得到示例代码的内部类Inner的字节码。
其中有：
```
final jvm.innerClass.Outter this$0;
  descriptor: Ljvm/innerClass/Outter;
  flags: ACC_FINAL, ACC_SYNTHETIC
```
ACC_SYNTHETIC表示是由编译器生成，而不是由用户编写的程序源代码经过编译器编译生成。
因此我们可以知道：
编译器会默认为成员内部类添加了一个指向外部类对象的引用。

而这个引用是如何赋初值的呢？
字节码中有这个片段:
```
jvm.innerClass.Outter$Inner(jvm.innerClass.Outter, jvm.innerClass.Outter$Inner);
  descriptor: (Ljvm/innerClass/Outter;Ljvm/innerClass/Outter$Inner;)V
  flags: ACC_SYNTHETIC
    Code:
      stack=2, locals=3, args_size=3
        0: aload_0
        1: aload_1
        2: invokespecial #45    // Method "<init>":(Ljvm/innerClass/Outter;)V
        5: return
      LineNumberTable:
        line 64: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
```

这是一个内部类的构造方法，虽然我们在代码中定义的是无参构造器，编译器还是会默认添加一个参数，该参数的类型为指向外部类对象的一个引用，所以成员内部类中的 this$0 指针便指向了外部类对象，因此可以在成员内部类中随意访问外部类的成员。

而这也间接说明了成员内部类是依赖于外部类的，如果没有创建外部类的对象，则无法对this$0引用进行初始化赋值，也就无法创建成员内部类的对象了。


那么为什么连 OutterClass 的 private 成员也可以访问呢？
通过查看 OutterClass 的字节码，我们发现:

```
static int access$0(jvm.innerClass.Outter);
  descriptor: (Ljvm/innerClass/Outter;)I
  flags: ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=1, locals=1, args_size=1
        0: aload_0
        1: getfield    #17        // Field i:I
        4: ireturn
      LineNumberTable:
        line 42: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
        
static int access$1(jvm.innerClass.Outter);
  descriptor: (Ljvm/innerClass/Outter;)I
  flags: ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=1, locals=1, args_size=1
        0: aload_0
        1: getfield    #19       // Field c:I
        4: ireturn
      LineNumberTable:
        line 43: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature        
```

access$0 和access$1 方法都是由编译器生成的，这2个方法分别用于返回外部类 Outter 的 “i”、“c” 这2个field。
所以，内部类中调用 System.out.println(i)，实际上运行的时候调用的是：
System.out.println(this$0.access$0(this$0));


#### 如果内部类只有一个private的构造函数，外部类还可以创建内部类的对象吗？
从示例代码可以看出，虽然内部类Inner的构造器是private，但是外部类Outter依然可以创建内部类的对象，为什么？
1. 首先编译器将外、内部类编译后放在同一个包中。
2. 编译器帮我们在内部类创建了包可见构造器，从而使外部类获得了创建权限。
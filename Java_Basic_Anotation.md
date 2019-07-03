---
layout: post
title: "Java注解"
date: 2018-05-12 08:55
comments: false
tags: 
- IO
- Java
categories:	
- 基础
- Java
---

本文首先展示一个比较简单的Java注解例子，然后分析这个注解的一些字节码。
<!--more-->

### 注解的示例

```
// MyTag注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MyTag {

  public enum LightStatus {ON , OFF}
  LightStatus Light() default LightStatus.OFF;
  
  String name() default "default_name";
  int age() default 18;
}


public class Demo {

  public static void m1() {}
  
  @MyTag
  public static void m2() {}
  
  @MyTag(name = "A")
  public static void m3() {}
  
  @MyTag(name = "B", age = 30)
  public static void m4() {}
  
  @MyTag(name = "C", age = 40, Light=LightStatus.ON)
  public static void m5() {}
  
}


// Util类
public class Util {

  public static void process(String clazz){
    Class targetClass = null;
    
    try {
      targetClass = Class.forName(clazz);
    }
    catch (ClassNotFoundException e) {
      e.printStackTrace();
    }
    
    for (Method m : targetClass.getMethods()) 
      if (m.isAnnotationPresent(MyTag.class)) {
        MyTag tag = m.getAnnotation(MyTag.class);
        System.out.println("方法" + m.getName() + "的MyTag注解内容为：" + tag.name() + "，" + tag.age()+"，"+tag.Light());
      }
  }
  
}
```
运行Util的process()方法分析Demo的注解，得到结果:
方法m2的MyTag注解内容为：default_name，18，OFF
方法m3的MyTag注解内容为：A，18，OFF
方法m4的MyTag注解内容为：B，30，OFF
方法m5的MyTag注解内容为：C，40，ON


### 相关的字节码

将会省略部分字节码。

#### MyTag
```
public interface MyTag extends java.lang.annotation.Annotation

  // 这是对于MyTag这个类的flag
  flags: ACC_PUBLIC, ACC_INTERFACE, ACC_ABSTRACT, ACC_ANNOTATION
  
  Constant pool:
    ...
    #10 = Utf8               default_name
    ...
    #13 = Integer            18
    ...
    #16 = Utf8               LMyTag$LightStatus;
    #17 = Utf8               OFF
    ...
    #21 = Utf8               Ljava/lang/annotation/Retention;
    #22 = Utf8               value
    #23 = Utf8               Ljava/lang/annotation/RetentionPolicy;
    #24 = Utf8               RUNTIME
    #25 = Utf8               Ljava/lang/annotation/Target;
    #26 = Utf8               Ljava/lang/annotation/ElementType;
    #27 = Utf8               METHOD
    #28 = Utf8               InnerClasses
    #29 = Class              #30    // jvm/annotation/MyTag$LightStatus
    #30 = Utf8               jvm/annotation/MyTag$LightStatus
    #31 = Utf8               LightStatus
  
  public abstract java.lang.String name();
    descriptor: ()Ljava/lang/String;
    flags: ACC_PUBLIC, ACC_ABSTRACT
    AnnotationDefault: default_value: s#10  //即“default_name”

  public abstract int age();
    descriptor: ()I
    flags: ACC_PUBLIC, ACC_ABSTRACT
    AnnotationDefault: default_value: I#13  //即 18
      
  public abstract MyTag$LightStatus Light();
    descriptor: ()LMyTag$LightStatus;
    flags: ACC_PUBLIC, ACC_ABSTRACT
    AnnotationDefault:
      default_value: e#16.#17}            // 即 MyTag$LightStatus.OFF

  RuntimeVisibleAnnotations:
    0: #21(#22=e#23.#24)      // Retention(value=RetentionPolicy.RUNTIME)
    1: #25(#22=[e#26.#27])    // Target(ElementType.METHOD)
  
  InnerClasses:
     public static final #31= #29 of #1; //LightStatus=class MyTag$LightStatus of class MyTag
```

#### Demo
我们只关注Demo的各个方法的字节码。
```
/* m1()方法没有加注解，列出来仅仅为了和其他方法对比 */
public static void m1();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    ... //Code部分省略
      
/* m2()方法加注解 “@MyTag” */      
public static void m2();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_STATIC
  RuntimeVisibleAnnotations:
    0: #17()                        //即 MyTag;
  Code:
    ... //Code部分省略    

/* m3()方法加注解 “@MyTag(name = "A")” */   
 public static void m3();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_STATIC
  RuntimeVisibleAnnotations:
    0: #17(#19=s#20)              //即 MyTag(name="A")
  Code:
    ... //Code部分省略  

/* m4()方法加注解 “@MyTag(name = "B", age = 30)” */         
public static void m4();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_STATIC
  RuntimeVisibleAnnotations:
    0: #17(#19=s#22,#23=I#24)      // 即 MyTag(name="B" , age=30)
  Code:
      ... //Code部分省略       
  
/* m5()方法加注解 “@MyTag(name = "C", age = 40, Light=LightStatus.ON)” */            
public static void m5();
  descriptor: ()V
  flags: ACC_PUBLIC, ACC_STATIC
  RuntimeVisibleAnnotations:
    0: #17(#19=s#26,#23=I#27,#28=e#29.#30)  // 即 MyTag(name="C" , age=40 , Light=MyTag$LightStatus.ON)
  Code:
    ... //Code部分省略        
```
不难理解，这里方法与注解的联系就在 RuntimeVisibleAnnotations 这个属性上面。

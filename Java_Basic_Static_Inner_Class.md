---
layout: post
title: "静态内部类"
date: 2018-5-04 22:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---


提出问题：
为什么静态内部类不能使用外部类的非static成员变量或者方法？

<!--more-->

先看以下示例代码:

```
class Outter{
　　private int i = 0;
　　private static int b = 1;
　　
　　public Outter() {}
　　public Inner inner = new Inner();
　　
　　private static class Inner {
　　　　public void f(){
　　　　　　b = 2;
　　　　　　//　不能访问变量ｉ
　　　　　　//	i = 0;
　　　　｝
　　｝
｝
```
查看示例代码Inner的字节码可以发现，它没有一个指向外围类对象的引用，即没有类似`final Outer this$0;`这样的引用。

所以我们可以得到结论：
静态内部类是不需要依赖于外部类的，这点和类的静态成员属性有点类似，并且它【不能使用外部类的非static成员变量或者方法】。
因为在没有外部类的对象的情况下，可以创建静态内部类的对象，如果允许访问外部类的非static成员就会产生矛盾，因为外部类的非static成员必须依附于具体的对象。
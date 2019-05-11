---
layout: post
title: "ArrayList的序列化"
date: 2017-11-02 21:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

ArrayList的定义如下：
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
  // ... 省略
  transient Object[] elementData;
}
```
留意到用于存储元素的数组使用了`transient`。<font color=red>为什么在ArrayList中使用transient修饰elementData？</font>


<!--more-->

### JDK序列化的API
当一个类不仅实现了Serializable接口，而且定义了 readObject(ObjectInputStream in)和writeObject(ObjectOutputStream out)方法，那么将按照如下的方式进行序列化和反序列化:

 * ObjectOutputStream会调用这个类的writeObject方法进行序列化
 * ObjectInputStream会调用相应的readObject方法进行反序列化。


### transient关键字
将不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会序列化到指定的目的地中。


### ArrayList的序列化
在ArrayList中定义的elementData:
```bash
private transient Object[] elementData;
```
再看ArrayList的writeObject方法：

```bash
private void writeObject(java.io.ObjectOutputStream s)
throws java.io.IOException{
  int expectedModCount = modCount;
  s.defaultWriteObject();

  // Write out size as capacity for behavioural compatibility with clone()
  s.writeInt(size);

  // Write out all elements in the proper order.
  for (int i=0; i<size; i++) {
    s.writeObject(elementData[i]);
  }

  if (modCount != expectedModCount) {
    throw new ConcurrentModificationException();
  }
}
```

### 为什么在ArrayList中使用transient修饰elementData？
1. ArrayList的自动扩容机制，elementData数组相当于容器，当容器不足时就会再扩充容量，但是容器的容量往往都是大于或者等于ArrayList所存元素的个数。
2. 所以ArrayList将elementData设计为transient，然后在writeObject方法中手动将其序列化，并且只序列化了实际存储的那些元素，而不是整个数组。
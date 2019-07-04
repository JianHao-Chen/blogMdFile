---
layout: post
title: "HotSpot中类与对象的表示"
date: 2019-06-04 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

HotSpot中类与对象的表示，使用所谓的***OOP-Klass 二分模型***，下面了解这部分的内容。

<!--more-->



### 对象表示机制

#### 对象的内存布局
HotSpot虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

##### 对象头
* Mark Word：用于存储对象自身的运行时数据，如哈希码（HashCode）、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳、对象分代年龄
 
* 元数据指针：指向描述类型的Klass对象的指针，Klass对象包含了实例对象所属类型的元数据（meta data），因此该字段称为元数据指针。虚拟机在运行时将频繁使用这个指针定位到位于方法区内的类型信息。

* 如果对象是一个 Java 数组：那在对象头中还必须有一块用于记录数组长度的数据。因为虚拟机可以通过普通 Java 对象的元数据信息确定 Java 对象的大小，但是从数组的元数据中无法确定数组的大小。

##### 实例数据
实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。
这部分的存储顺序会受到虚拟机分配策略参数（FieldsAllocationStyle）和字段在 Java 源码中定义顺序的影响。

##### 对齐填充 
对齐填充不是必然存在的，没有特别的含义，它仅起到占位符的作用。

由于 HotSpot VM 的自动内存管理系统要求对象起始地址必须是 8 字节的整数倍，也就是说对象的大小必须是 8 字节的整数倍。对象头部分是 8 字节的倍数，所以当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

#### OOP-Klass 二分模型
OOP-Klass 二分模型在HotSpot内部用于表示Java对象。
* OOP：ordinary object pointer，普通对象指针，用来描述对象实例信息。
* Klass：Java类的c++对等体，用来描述Java类。

<font color=red> **这样设计的原因** ：</font><font color=blue>***没必要令每一个对象都持有任何虚函数！***</font>

##### 模块划分
这部分属于 vm下的 oops 模块。而oops 模块的组成如下：
![](/assets/blogImg/JVM/Oop_Klass/1.png)


##### 小结
 
设计这套模型的原因在代码的注释有说明：
```
// A Klass is the part of the klassOop that provides:
//  1: language level class object (method dictionary etc.)
//  2: provide vm dispatch behavior for the object
// Both functions are combined into one C++ class. The toplevel class "Klass"
// implements purpose 1 whereas all subclasses provide extra virtual functions
// for purpose 2.

// One reason for the oop/klass dichotomy in the implementation is
// that we don't want a C++ vtbl pointer in every object.  Thus,
// normal oops don't have any virtual functions.  Instead, they
// forward all "virtual" functions to their klass, which does have
// a vtbl and does the C++ dispatch depending on the object's
// actual type.  (See oop.inline.hpp for some of the forwarding code.)
// ALL FUNCTIONS IMPLEMENTING THIS DISPATCH ARE PREFIXED WITH "oop_"!
```
其实就是<font color=red>不想让每个对象都包含vtbl(虚方法表)，其中oop中不含有任何虚函数，虚函数表保存于klass中，可以进行method dispatch</font>。



### 相关oop和Klass 的内存布局

#### 相关的 oop 结构：
```
// oopDesc is the top baseclass for objects classes.
class oopDesc {
    private:
        volatile markOop  _mark;            
        union _metadata {
        wideKlassOop    _klass;
        narrowOop       _compressed_klass;
    } _metadata;
    ....
}
  

// 通常使用 instanceOopDesc 来表示一个 Java对象
class instanceOopDesc : public oopDesc {
    // 没有其他变量
}


// 是所有 array 类型的抽象基类
// 这个类没有定义纯虚函数来保证它是抽象类，因为这样会使这个c++类的每一个实例都会分配虚函数表

// The layout of array Oops is:

//  markOop
//  klassOop  // 32 bits if compressed but declared 64 in LP64.
//  length    // shares klass memory or allocated after declared fields.
class arrayOopDesc : public oopDesc {

    /**   计算 header size : length字段的 offset +  length这个字段本身的大小（一个int）  */
    // Header size computation.
    // The header is considered the oop part of this type plus the length.
    // Returns the aligned header_size_in_bytes.  This is not equivalent to
    // sizeof(arrayOopDesc) which should not appear in the code.
    static int header_size_in_bytes() {
        size_t hs = align_size_up(length_offset_in_bytes() + sizeof(int),
                              HeapWordSize);

    public:
    /**  计算 length字段的 offset  */
    // The _length field is not declared in C++.  It is allocated after the
    // declared nonstatic fields in arrayOopDesc if not compressed, otherwise
    // it occupies the second half of the _klass field in oopDesc.
    static int length_offset_in_bytes() {
        return UseCompressedOops ? klass_gap_offset_in_bytes() :
                               sizeof(arrayOopDesc);
    }
  
      // ... 省略
}
```
* _mark是markOop类型对象，用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等等，占用内存大小与虚拟机位长一致。
* _metadata是一个结构体，wideKlassOop（64位）和narrowOop（32位）都用于指向描述类型的Klass对象的指针（其中narrowOop是经过压缩的指针），Klass对象包含了实例对象所属类型的元数据（meta data），因此该字段称为元数据指针。虚拟机在运行时将频繁使用这个指针定位到位于方法区内的类型信息。
* `arrayOopDesc`的_length字段并没有定义在c++，而是根据VM参数“UseCompressedOops”而定：
  * 如果没有开启压缩OOPS（-XX:-UseCompressedOops）,length字段分配在整个arrayOopDesc结构体之后
  * 如果开启压缩OOPS（-XX:+UseCompressedOops）,length字段分配在union`_metadata`的_klass的第二部分，因为此时存储的指针只占32位。
  


对象和数组的内存布局如下图：
![](/assets/blogImg/JVM/Oop_Klass/2.png)
![](/assets/blogImg/JVM/Oop_Klass/3.png)

#### 相关的 Klass 结构：

##### 注意
以`instanceKlass`为例，，由于HotSpot从 JDK 1.8 开始移除了“PerGen(永生代)”，所以 `instanceKlass` 的结构不再包含 oop。


##### Klass
![](/assets/blogImg/JVM/Oop_Klass/4.png)

##### instanceKlass
![](/assets/blogImg/JVM/Oop_Klass/5.png)





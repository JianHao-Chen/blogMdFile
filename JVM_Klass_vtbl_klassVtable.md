---
layout: post
title: "Klass_vtbl和klassVtable的区别"
date: 2019-06-05 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

看JVM代码过程中，看到`Klass_vtbl`和`klassVtable`，不明白，于是研究下它们的区别。

<!--more-->


### klassVtable

代码如下：
```
// A klassVtable abstracts the variable-length vtable that is embedded in instanceKlass
// and arrayKlass.  klassVtable objects are used just as convenient transient accessors to the vtable,
// not to actually hold the vtable data.
// Note: the klassVtable should not be accessed before the class has been verified
// (until that point, the vtable is uninitialized).

// Currently a klassVtable contains a direct reference to the vtable data, and is therefore
// not preserved across GCs.

class klassVtable : public ResourceObj {
    KlassHandle  _klass;            // my klass
    int          _tableOffset;      // offset of start of vtable data within klass
    int          _length;           // length of vtable (number of entries)

    // ...
}
```
* 这里说的`vtable`是存在于`instanceKlass`或`arrayKlass`的，用于实现Java多态的。
* HotSpot并没有为上面的`vtable`定义数据结构，而是直接在这些“XXKlass”的内存后面存放vtable的数据
* `klassVtable`是一个对`vtable`数据的“访问器”。


### Klass_vtbl

`Klass_vtbl`定义在文件klass.hpp，代码如下：
```
// Holder (or cage) for the C++ vtable of each kind of Klass.
// We want to tightly constrain the location of the C++ vtable in the overall layout.
class Klass_vtbl {

  protected:
    // The following virtual exists only to force creation of a C++ vtable,
    // so that this class truly is the location of the vtable of all Klasses.
    virtual void unused_initial_virtual() { }
  
  public:
    // ... 省略很长的一段的注释
    virtual void* allocate_permanent(KlassHandle& klass, int size, TRAPS) const = 0;
    void post_new_init_klass(KlassHandle& klass, klassOop obj, int size) const;
}  
```
* `Klass_vtbl`没有成员变量，还定义了一个虚方法。
* `Klass`继承`Klass_vtbl`。
* 按照C++的规则，上面2点使得每一个 Klass 对象一定有一个 _vptr（指向虚函数表的指针），并且这个 _vptr 在内存的位置在`Klass`定义的变量之前。
* 至于为什么要强制生成这个 _vptr，应该是想把这个 _vptr “固定下来”，不想 XKlass 有而 YKlass 没有 _vptr，那么在后面计算vtable位置时就不一致了。（待核实）






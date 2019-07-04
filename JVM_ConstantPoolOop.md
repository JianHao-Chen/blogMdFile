---
layout: post
title: "HotSpot的运行时常量池"
date: 2019-06-12 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

在HotSpot里面，是使用 constantPoolOop 表示一个Java运行时常量池的。下面介绍 constantPoolOop。

<!--more-->

### 运行时常量池

#### 运行时常量池的意义
* 运行时常量池保存的是它所关联的类用到的类、字段、方法的信息，可以说是这个类的==资源仓库==。
* 字节码使用常量池项的例子如`invokevirtual #5`，通过常量池的索引，字节码就能访问对应的方法，既==不破坏指令集的简洁性==，又能定位到目标。
* Java代码编译后，在常量池项的有些类、方法、字段是以==符号引用==来表示的（即只有一个符号，没有真正的内存地址，需要在后续解析，把真正的地址放到常量池项里面）。这个==对于动态连接（多态）是很重要的==。


#### 运行时常量池的创建

##### 位置
JVM规定：
> 运行时常量池在运行期间在方法区中分配

而在当前版本，运行时常量池是在永久代（方法区在永久代实现）。

##### 时机

HotSpot在解析class文件时，会解析class文件的常量池(`ClassFileParser::parse_constant_pool`)，并且返回运行时常量池。通过`oopFactory::new_constantPool`创建`constantPoolOop`(这是运行时常量池在HotSpot的数据结构)：
```
/**
 *  通过全局唯一的 constantPoolKlass 对象的 allocate方法进行 constantPoolOop的创建
*/
constantPoolOop oopFactory::new_constantPool(int length,
                                             bool is_conc_safe,
                                             TRAPS) {
    constantPoolKlass* ck = constantPoolKlass::cast(Universe::constantPoolKlassObj());
    return ck->allocate(length, is_conc_safe, CHECK_NULL);
}


/**
 *  (1) 在永久代为 constantPoolOop 分配内存
 *  (2) 初始化 constantPoolOop
*/
constantPoolOop constantPoolKlass::allocate(int length, bool is_conc_safe, TRAPS) {
    int size = constantPoolOopDesc::object_size(length);
    KlassHandle klass (THREAD, as_klassOop());
    assert(klass()->is_oop(), "Can't be null, else handlizing of c below won't work");
    
    constantPoolHandle pool;
    {
      constantPoolOop c =
        (constantPoolOop)CollectedHeap::permanent_obj_allocate(klass, size    CHECK_NULL);
      assert(c->klass_or_null() != NULL, "Handlizing below won't work");
      pool = constantPoolHandle(THREAD, c);
    }
    
    pool->set_length(length);
    pool->set_tags(NULL);
    pool->set_cache(NULL);
    pool->set_operands(NULL);
    pool->set_pool_holder(NULL);
    pool->set_flags(0);
    pool->set_orig_length(0);
    pool->set_is_conc_safe(is_conc_safe);
    assert(pool->is_oop() && pool->is_parsable(), "Else size() below is unreliable");
    assert(size == pool->size(), "size() is wrong")    
    // initialize tag array
    typeArrayOop t_oop = oopFactory::new_permanent_byteArray(length, CHECK_NULL);
    typeArrayHandle tags (THREAD, t_oop);
    for (int index = 0; index < length; index++) {
      tags()->byte_at_put(index, JVM_CONSTANT_Invalid);
    }
    pool->set_tags(tags())    
    // Check that our size was stable at its old value.
    assert(size == pool->size(), "size() changed");
    return pool();
}
```



#### 运行时常量池的结构

![](/assets/blogImg/JVM/ConstantPool/1.png)


注意：
CPSlot数组没有显式定义在常量池的结构中，而是通过指针操作直接读写。


##### constantPoolOopDesc的定义
```
class constantPoolOopDesc : public oopDesc {
  private:
    typeArrayOop         _tags; // the tag array describing the constant pool's contents
    constantPoolCacheOop _cache;         // the cache holding interpreter runtime information
    klassOop             _pool_holder;   // the corresponding class
    typeArrayOop         _operands;      // for variable-sized (InvokeDynamic) nodes, usually empty
    int                  _flags;         // a few header bits to describe contents for GC
    int                  _length; // number of elements in the array
    volatile bool        _is_conc_safe; // if true, safe for concurrent
                                      // GC processing
    // only set to non-zero if constant pool is merged by RedefineClasses
    int                  _orig_length;
    
    
    // ... 省略，只列出对 int 类型在运行时常量池的读写操作
    
    intptr_t* base() const { return (intptr_t*) (((char*) this) + sizeof(constantPoolOopDesc)); }
    oop* tags_addr()       { return (oop*)&_tags; }
    oop* cache_addr()      { return (oop*)&_cache; }
    oop* operands_addr()   { return (oop*)&_operands; }
    
    
    // Tells whether index is within bounds.
    bool is_within_bounds(int index) const {
        return 0 <= index && index < length();
    }
    
    jint* int_at_addr(int which) const {
        assert(is_within_bounds(which), "index out of bounds");
        return (jint*) &base()[which];
    }
    
    
    
  public:
    void int_at_put(int which, jint i) {
        tag_at_put(which, JVM_CONSTANT_Integer);
        *int_at_addr(which) = i;
    }
 
    jint int_at(int which) {
        assert(tag_at(which).is_int(), "Corrupted constant pool");
        return *int_at_addr(which);
    }
}
```

##### 使用运行时常量池的缺点
如果字节码每次都要先解析常量池项，性能会下降，于是就有常量池缓存了。


### 常量池缓存
又叫常量池Cache。为Java的类、方法、字段提供快速访问的入口。

常量池缓存由一个数组组成，每一个元素是常量池缓存项，每一个缓存项表示类中引用的一个字段或方法，因此缓存项有以下2种：
* 字段项：用来支持对类变量和对象的快速访问。
* 方法项：用来支持invoke系列函数调用指令，为这些指令提供快速定位目标方法的能力。

HotSpot用`ConstantPoolCacheEntry`实现常量池缓存项（在cpCacheOop.hpp里面）。
它的结构如下：
```
// bit number |31                0|
// bit length |-8--|-8--|---16----|
// --------------------------------
// _indices   [ b2 | b1 |  index  ]
// _f1        [  entry specific   ]
// _f2        [  entry specific   ]
// _flags     [t|f|vf|v|m|h|unused|field_index] (for field entries)
// bit length |4|1|1 |1|1|0|---7--|----16-----]
// _flags     [t|f|vf|v|m|h|unused|eidx|psze] (for method entries)
// bit length |4|1|1 |1|1|1|---7--|-8--|-8--]
```

简单介绍jvm是如何使用`ConstantPoolCacheEntry`的：
* 对于`invokespecial`和`invokestatic`指令，f2字段表示目标函数的methodOop。
* 对于`invokevirtual`指令：
  * 如果是virtual final 方法，f2字段保存目标函数的methodOop
  * 否则（意味着用到了vtable），f2字段存放的是目标函数在vtable的索引编号。
* 对于`invokeinterface`指令，f1字段表示相应的klassOop，f2表示方法位于itable的索引编号。


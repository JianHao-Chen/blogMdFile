---
layout: post
title: "HotSpot对象的创建"
date: 2019-06-05 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文记录在研究**HotSpot对象创建**时的知识点和不懂的地方。

<!--more-->


### 字节码 new 的格式
```
--------------
|     new     |
--------------
|  indexbyte1 |
--------------
|  indexbyte2 |
--------------
```


通过 `(indexbyte1 << 8) | indexbyte2` , 作为常量池中 class 的索引。



### 字节码 new 的实现

在`bytecodeInterpreter.cpp`：
```
CASE(_new): {
    // 获取在 new 字节码后面的数值，这是对象类型信息在常量池的索引
    u2 index = Bytes::get_Java_u2(pc+1);
    constantPoolOop constants = istate->method()->constants();
    // 如果已经解析过
    if (!constants->tag_at(index).is_unresolved_klass()) {
        // Make sure klass is initialized and doesn't have a finalizer
        // 获得常量池项
        oop entry = constants->slot_at(index).get_oop();
        assert(entry->is_klass(), "Should be resolved klass");
        klassOop k_entry = (klassOop) entry;
        assert(k_entry->klass_part()->oop_is_instance(), "Should be instanceKlass");
        instanceKlass* ik = (instanceKlass*) k_entry->klass_part();
        
        // 快速分配
        if ( ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
            // 先尝试在 TLAB分配
            size_t obj_size = ik->size_helper();
            oop result = NULL;
            // If the TLAB isn't pre-zeroed then we'll have to do it
            bool need_zero = !ZeroTLAB;
            if (UseTLAB) {
              result = (oop) THREAD->tlab().allocate(obj_size);
            }
            // TLAB分配不成功，则在 Eden 区分配。
            if (result == NULL) {
              need_zero = true;
              // Try allocate in shared eden
        retry:
              HeapWord* compare_to = *Universe::heap()->top_addr();
              HeapWord* new_top = compare_to + obj_size;
              if (new_top <= *Universe::heap()->end_addr()) {
                if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) {
                  goto retry;
                }
                result = (oop) compare_to;
              }
            }
            if (result != NULL) {
                // Initialize object (if nonzero size and need) and then the header
                if (need_zero) {
                  HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) /       oopSize;
                  obj_size -= sizeof(oopDesc) / oopSize;
                  if (obj_size > 0 ) {
                    memset(to_zero, 0, obj_size * HeapWordSize);
                  }
                }
                if (UseBiasedLocking) {
                  result->set_mark(ik->prototype_header());
                } else {
                  result->set_mark(markOopDesc::prototype());
                }
                result->set_klass_gap(0);
                result->set_klass(k_entry);
                SET_STACK_OBJECT(result, 0);
                UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
            }
        }
    }
        
    // Slow case allocation（慢速分配）
    CALL_VM(InterpreterRuntime::_new(THREAD, METHOD->constants(), index),
                handle_exception);
    SET_STACK_OBJECT(THREAD->vm_result(), 0);
    THREAD->set_vm_result(NULL);
    UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);

}
```

#### new 字节码流程的总结：
##### 分配内存
* 根据new字节码后面的索引值，取出常量池中的对象类型信息。
* 如果此类已经被加载和解析，那么采用“快速分配”，否则使用“慢速分配”。
* **快速分配**:只在内存空间中划分可用内存，因此可以较高效的完成实例分配。同时，根据分配内存空间的来源，可以分为：
  * 使用TLAB（ThreadLocal Allocation Buffer）
  * 使用Eden空间
* **慢速分配**:需要先进行类的解析、初始化。
* TLAB是线程私有空间，分配空间不需要加锁。如果分配失败，会通过加锁在Eden区分配
* Eden区是所有线程共享的，需要保证线程安全。这里采用 ***原子操作 + retry*** 的方式，直到分配成功。

##### 初始化零值
JVM还会对分配好的实例空间进行赋初值（填零）

##### 设置对象头
* 设置 mark_word
* 设置 类型元数据指针

##### ？？设置栈顶对象引用 ??（存在疑问）
设置栈顶对象引用

<font color=red>【问题】<init>方法是什么时候调用呢？</font>
上面 new字节码 的代码中，并没有调用对象的构造器方法 <init>，就把新建对象的引用压到栈顶。


---
layout: post
title: "HotSpot方法的表示"
date: 2019-06-12 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

在HotSpot里面，是使用 methodOop 表示一个Java方法的。下面介绍 methodOop。

<!--more-->



#### methodOop的创建

##### 创建时机
在解析class文件时(`ClassFileParser::parseClassFile`)，调用了`parse_methods`方法，这个方法会解析class文件的所有方法和方法上的注解。

在`ClassFileParser::parse_method`方法里面，当size相关的信息准备好，就创建一个 methodOop对象：
```
// All sizing information for a methodOop is finally available, now create it
methodOop m_oop  = oopFactory::new_method(code_length, access_flags, linenumber_table_length,
                                            total_lvt_length, checked_exceptions_length,
                                            oopDesc::IsSafeConc, CHECK_(nullHandle));
```


##### 创建位置
methodOop的内存是从PermGen(永久代)分配的。
`oopFactory::new_method`里面会调用`methodKlass::allocate`方法：
```
methodOop methodKlass::allocate(constMethodHandle xconst,
                                AccessFlags access_flags, TRAPS) {
                                
  int size = methodOopDesc::object_size(access_flags.is_native());
  KlassHandle h_k(THREAD, as_klassOop());
  assert(xconst()->is_parsable(), "possible publication protocol violation");
  
  //在永久代分配
  methodOop m = (methodOop)CollectedHeap::permanent_obj_allocate(h_k, size, CHECK_NULL);
  assert(!m->is_parsable(), "not expecting parsability yet.");

  No_Safepoint_Verifier no_safepoint;  // until m becomes parsable below
  
  // 赋初值
  m->set_constMethod(xconst());
  m->set_access_flags(access_flags);
  m->set_method_size(size);
  m->set_name_index(0);
  m->set_signature_index(0);
  // ... 省略
}
```


#### methodOop的实现
```
class methodOopDesc : public oopDesc {

  private:
  
    // 方法只读数据
    constMethodOop    _constMethod;                // Method read-only data.
    // 常量池
    constantPoolOop   _constants;                  // Constant pool
    methodDataOop     _method_data;
    // 解析器调用次数
    int               _interpreter_invocation_count; // Count of times invoked (reused as prev_event_count in tiered)
    // 访问标识
    AccessFlags       _access_flags;               // Access flags
    // 该methodOop在vtable表中的索引位置
    int               _vtable_index;               // vtable index of this method (see VtableIndexFlag)
                                                   // note: can have vtables with >2**16 elements (because of inheritance)
    // 占用大小                                               
    u2                _method_size;                // size of this object
    // 操作数栈最大元素个数
    u2                _max_stack;                  // Maximum number of entries   on the expression stack
    // 局部变量最大元素个数
    u2                _max_locals;                 // Number of local variables   used by this method
    // 参数块占用的大小
    u2                _size_of_parameters;         // size of the parameter   block (receiver + arguments) in words
    // 
    u1                _intrinsic_id;               // vmSymbols::intrinsic_id   (0 == _none)
    // 解析运行时以异常方式退出方法的次数
    u2                _interpreter_throwout_count; // Count of times method was   exited via exception while interpreting
    u2                _number_of_breakpoints;      // fullspeed debugging   support   
    
    // 计数器，统计方法或循环体的被调用次数，用于基于触发频率的优化
    InvocationCounter _invocation_counter;         // Incremented before each activation of the method - used to trigger frequency-based optimizations
    InvocationCounter _backedge_counter;           // Incremented before each backedge taken - used to trigger frequencey-based optimizations


    // Entry point for calling both from and to the interpreter.
    // 解析器调用入口地址
    address _i2i_entry;           // All-args-on-stack calling convention
    // Adapter blob (i2c/c2i) for this methodOop. Set once when method is   linked.
    AdapterHandlerEntry* _adapter;
    
    // Entry point for calling from compiled code, to compiled code if it   exists
    // or else the interpreter.
    // 编译代码入口
    volatile address _from_compiled_entry;        // Cache of: _code ?   _code->entry_point() : _adapter->c2i_entry()
    
    // The entry point for calling both from and to compiled code is
    // "_code->entry_point()".  Because of tiered compilation and de-opt, this
    // field can come and go.  It can transition from NULL to not-null at any
    // time (whenever a compile completes).  It can transition from not-null to
    // NULL only at safepoints (because of a de-opt).
    // 指向本地代码
    nmethod* volatile _code;                       // Points to thecorresponding piece of native code
    volatile address           _from_interpreted_entry; // Cache of _code ?_adapter->i2c_entry() : _i2i_entry


}

```


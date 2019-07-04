---
layout: post
title: "HotSpot的类加载--Loading"
date: 2019-06-09 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---





在学习HotSpot的类加载部分源码的过程中，写下以下几篇文章：
* HotSpot的类加载--概述
* HotSpot的类加载--Loading（本文）
* HotSpot的类加载--Linking
* HotSpot的类加载--Initialization

<!--more-->

### 加载 (Loading)

#### 定义
加载是类加载过程中的一个阶段，这个阶段会在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的入口。


#### Classfile模块
HotSpot的加载功能是由Classfile模块提供的。它包含以下模块：
* ClassFileParser： 类解析器，用来解析 *.class文件
* Verifier：验证器，用来验证*.class文件中的字节码。它将为每一个类创建一个ClassVerifier实例来验证。
* CLassLoader：类加载器。
* SystemDictionary：系统字典，用来记录已加载的所有类。
* SymboleTable：字符表，用作快速查询字符串，例如将与JDK基本类的名字相映射的字符串、表示函数签名类型的字符串以及VM内部各种用途的字符串等。


#### 加载类的主要过程
这个过程在方法`SystemDictionary::resolve_from_stream`中：
```
// Add a klass to the system from a stream (called by jni_DefineClass and
// JVM_DefineClass).
// Note: class_name can be NULL. In that case we do not know the name of
// the class until we have parsed the stream.

klassOop SystemDictionary::resolve_from_stream(Symbol* class_name,
                                               Handle class_loader,
                                               Handle protection_domain,
                                               ClassFileStream* st,
                                               bool verify,
                                               TRAPS) {

  //【1. 加锁】
  // Make sure we are synchronized on the class loader before we proceed
  Handle lockObject = compute_loader_lock_object(class_loader, THREAD);
  check_loader_lock_contention(lockObject, THREAD);
  ObjectLocker ol(lockObject, THREAD, DoObjectLock);
    
  // 【2. 对Class文件的数据流进行解析，生成instanceKlassHandle对象】
  instanceKlassHandle k = ClassFileParser(st).parseClassFile(class_name,
                                                             class_loader,
                                                             protection_domain,
                                                             parsed_name,
                                                             verify,
                                                             THREAD);
                                                             
  // 【3.进行一些检查，如不能在‘java’包下，类名不能包含‘.’】
  //  这里的代码省略
  
  
  return k();
}                                               
```
上面代码主要是通过调用`ClassFileParser`模块完成其中最重要的类文件解析工作的。而`ClassFileParser`又是通过`ClassFileStream`读取Class文件字节流的。

`parseClassFile`方法如下，省略部分代码：
```
instanceKlassHandle ClassFileParser::parseClassFile(Symbol* name,
                                                    Handle class_loader,
                                                    Handle protection_domain,
                                                    KlassHandle host_klass,
                                                    GrowableArray<Handle>* cp_patches,
                                                    TempNewSymbol& parsed_name,
                                                    bool verify,
                                                    TRAPS) {

    // 获取流对象
    ClassFileStream* cfs = stream();
    
    // 【1】 Magic value
    u4 magic = cfs->get_u4_fast();
    guarantee_property(magic == JAVA_CLASSFILE_MAGIC,
                     "Incompatible magic value %u in class file %s",
                     magic, CHECK_(nullHandle));
    
    // 【2】 Version numbers
    u2 minor_version = cfs->get_u2_fast();
    u2 major_version = cfs->get_u2_fast();
    if (!is_supported_version(major_version, minor_version)) {
        // throw exception
    }
    _major_version = major_version;
    _minor_version = minor_version;
    
    // 【3】Constant pool
    constantPoolHandle cp = parse_constant_pool(CHECK_(nullHandle));
    ConstantPoolCleaner error_handler(cp); // set constant pool to be cleaned up.
    
    // 【4】Access flags
    AccessFlags access_flags;
    jint flags = cfs->get_u2_fast() & JVM_RECOGNIZED_CLASS_MODIFIERS;
    verify_legal_class_modifiers(flags, CHECK_(nullHandle));
    access_flags.set_flags(flags);
    
    // 【5】This class and superclass
    instanceKlassHandle super_klass;
    u2 this_class_index = cfs->get_u2_fast();
    Symbol*  class_name  = cp->unresolved_klass_at(this_class_index);
    u2 super_class_index = cfs->get_u2_fast();
    super_klass = instanceKlassHandle(THREAD, cp->resolved_klass_at(super_class_index));
    
    // 【6】 Interfaces
    u2 itfs_len = cfs->get_u2_fast();
    objArrayHandle local_interfaces;
    if (itfs_len == 0) {
      local_interfaces = objArrayHandle(THREAD, Universe::the_empty_system_obj_array());
    } else {
      local_interfaces = parse_interfaces(cp, itfs_len, class_loader, protection_domain, _class_name, CHECK_(nullHandle));
    }
    
    
    // 【7】Fields (offsets are filled in later)
    struct FieldAllocationCount fac = {0,0,0,0,0,0,0,0,0,0};
    objArrayHandle fields_annotations;
    typeArrayHandle fields = parse_fields(cp, access_flags.is_interface(), &fac, &fields_annotations, CHECK_(nullHandle));
    
    // 【8】 Methods
    objArrayOop methods_annotations_oop = NULL;
    objArrayOop methods_parameter_annotations_oop = NULL;
    objArrayOop methods_default_annotations_oop = NULL;
    objArrayHandle methods = parse_methods(cp, access_flags.is_interface(),
                                           &promoted_flags,
                                           &has_final_method,
                                           &methods_annotations_oop,
                                           &methods_parameter_annotations_oop,
                                           &methods_default_annotations_oop,
                                           CHECK_(nullHandle));
    
    
    // 【9】 Size of vtable and itable
    int vtable_size = 0;
    int itable_size = 0;
    int num_miranda_methods = 0;
    
    klassVtable::compute_vtable_size_and_num_mirandas(vtable_size,
                                                      num_miranda_methods,
                                                      super_klass(),
                                                      methods(),
                                                      access_flags,
                                                      class_loader,
                                                      class_name,
                                                      local_interfaces(),
                                                      CHECK_(nullHandle));
    
    itable_size = access_flags.is_interface() ? 0 : klassItable::compute_itable_size(transitive_interfaces);
    
    
    // 【10】创建当前类instanceKlass，并且将已解析的值赋值进去。
    klassOop ik = oopFactory::new_instanceKlass(name, vtable_size, itable_size,
                                                static_field_size,
                                                total_oop_map_count,
                                                rt, CHECK_(nullHandle));
    instanceKlassHandle this_klass (THREAD, ik);
    
    // Fill in information already parsed
    this_klass->set_access_flags(access_flags);
    this_klass->set_should_verify_class(verify);
    // 。。。省略
    
    
    // 【11】通知类已加载，更新PerfData计数器
    ClassLoadingService::notify_class_loaded(instanceKlass::cast(this_klass()),
                                             false /* not shared class */);
    
    return this_klass;                                         
}                                                    
```

上面代码的流程总结为下图：

![](/assets/blogImg/JVM/Class_Loading/Loading/1.png)





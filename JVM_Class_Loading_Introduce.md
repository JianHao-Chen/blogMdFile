---
layout: post
title: "HotSpot的类加载--概述"
date: 2019-06-08 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

在学习HotSpot的类加载部分源码的过程中，写下以下几篇文章：
* HotSpot的类加载--概述（本文）
* HotSpot的类加载--Loading
* HotSpot的类加载--Linking
* HotSpot的类加载--Initialization


<!--more-->

### Java类加载
Java类加载包括了5部分：加载，验证，准备，解析，初始化，如下图：
![](/assets/blogImg/JVM/Class_Loading/Introduce/1.png)

### 加载（Loading）

在 JDK的 `ClassLoader` 类里面有 `defineClass` 方法，提供类加载的功能，它依赖于 native方法 `defineClass1`（实现在 `ClassLoader.c` ）。
`defineClass1`最后调用了 jvm.cpp 的 `jvm_define_class_common` 方法：
```
static jclass jvm_define_class_common(JNIEnv *env, const char *name,
                                      jobject loader, const jbyte *buf,
                                      jsize len, jobject pd, const char *source,
                                      jboolean verify, TRAPS) {
    // ... 省略
    
  TempNewSymbol class_name = NULL;
  if (name != NULL) {
    const int str_len = (int)strlen(name);
    if (str_len > Symbol::max_length()) {
      // It's impossible to create this class;  the name cannot fit
      // into the constant pool.
      THROW_MSG_0(vmSymbols::java_lang_NoClassDefFoundError(), name);
    }
    class_name = SymbolTable::new_symbol(name, str_len, CHECK_NULL);
  }
  
  ResourceMark rm(THREAD);
  ClassFileStream st((u1*) buf, len, (char *)source);
  Handle class_loader (THREAD, JNIHandles::resolve(loader));
  if (UsePerfData) {
    is_lock_held_by_thread(class_loader,
                           ClassLoader::sync_JVMDefineClassLockFreeCounter(),
                           THREAD);
  }
  Handle protection_domain (THREAD, JNIHandles::resolve(pd));
  klassOop k = SystemDictionary::resolve_from_stream(class_name, class_loader,
                                                     protection_domain, &st,
                                                     verify != 0,
                                                     CHECK_NULL);
                                      
  return (jclass) JNIHandles::make_local(env, Klass::cast(k)->java_mirror());
}                                      
```

**小结**：
1. 类加载（Loading）的方法调用链如下：
    ```
    ClassLoader::loadClass  --> ... --> ClassLoader::defindClass1  -->
    【native】ClassLoader::Java_java_lang_ClassLoader_defineClass1 --> ... -->
    jvm::jvm_define_class_common --> SystemDictionary::resolve_from_stream
    ```
2. `SystemDictionary`模块下的`resolve_from_stream`方法就是类加载(Loading)的核心。



### 连接（Linking）

找了JVM 对于连接的规定，大概有以下几点：
* 一个类（接口）一定按照 <font color=purple>加载（Loading）、验证、准备、初始化</font>这样的顺序进行的，但是 ***解析*** 阶段是特殊的。
* JVM规范没有规定解析的时间，有2种实现方式：
  * **早解析、静态解析** ：在验证时，一次性解析此类（接口）的所有符号引用
  * **晚解析** ：当这个符号引用被第一次使用时，才去解析。（HotSpot使用这种方式）



### 初始化（Initialization）
初始化阶段是执行类构造器的过程。

JVM规范规定有且仅有5种情况，立即对类进行初始化：
1. 遇到 new、getstatic、putstatic、invokestatic这4条字节码指令，并且类没有初始化
2. 使用 java.lang.reflect 包的方法对类进行反射调用时，并且类没有初始化
3. 初始化一个类时，如果它的父类没有初始化，则先对父类初始化
4. JVM启动时，用户指定一个执行的主类，JVM会先初始化这个类
5. 这个和 JDK 1.7 的动态语言支持相关




### 类加载过程的状态迁移

在HotSpot里面，类加载时共定义了以下几种状态：
```
// See "The Java Virtual Machine Specification" section 2.16.2-5 for a detailed description
// of the class loading & initialization procedure, and the use of the states.
enum ClassState {
    unparsable_by_gc = 0,               // object is not yet parsable by gc. Value of _init_state at object allocation.
    allocated,                          // allocated (but not yet linked)
    loaded,                             // loaded and inserted in class hierarchy (but not linked yet)
    linked,                             // successfully linked/verified (but not initialized yet)
    being_initialized,                  // currently running class initializer
    fully_initialized,                  // initialized (successfull final state)
    initialization_error                // error happened during initialization
};
```

以下是状态迁移图：
![](/assets/blogImg/JVM/Class_Loading/Introduce/2.png)







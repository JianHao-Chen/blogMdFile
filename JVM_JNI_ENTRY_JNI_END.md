---
layout: post
title: "JNI_ENTRY和JNI_END宏"
date: 2019-06-03 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文介绍 JVM 代码里面的`JNI_ENTRY`和`JNI_END`宏。
<!--more-->


在jni.cpp文件里面，有不少函数是使用了`JNI_ENTRY`和`JNI_END`宏的，一开始阅读起来让人很不习惯。

例如：

```
JNI_ENTRY(void, jni_SetStaticObjectField(JNIEnv *env, jclass clazz, jfieldID fieldID, jobject value))
  JNIWrapper("SetStaticObjectField");
 
  JNIid* id = jfieldIDWorkaround::from_static_jfieldID(fieldID);
  if (JvmtiExport::should_post_field_modification()) {
    jvalue field_value;
    field_value.l = value;
    JvmtiExport::jni_SetField_probe(thread, NULL, NULL, id->holder(), fieldID, true, 'L', (jvalue *)&field_value);
  }
  id->holder()->java_mirror()->obj_field_put(id->offset(), JNIHandles::resolve(value));
 
JNI_END
```


#### JNI_END
JNI_END的定义很简单：
```
// Close the routine and the extern "C"
#define JNI_END } }
```


#### JNI_ENTRY

JNI_ENTRY的定义（在 interfaceSupport.hpp）：
```
#define JNI_ENTRY(result_type, header)                               \
    JNI_ENTRY_NO_PRESERVE(result_type, header)                       \
    WeakPreserveExceptionMark __wem(thread);
    
    
#define JNI_ENTRY_NO_PRESERVE(result_type, header)             \
extern "C" {                                                         \
  result_type JNICALL header {                                \
    JavaThread* thread=JavaThread::thread_from_jni_environment(env); \
    assert( !VerifyJNIEnvThread || (thread == Thread::current()), "JNIEnv is only valid in same thread"); \
    ThreadInVMfromNative __tiv(thread);                              \
    debug_only(VMNativeEntryWrapper __vew;)                          \
    VM_ENTRY_BASE(result_type, header, thread)    
```

#### JNIWrapper
用于加入一些跟踪jni方法的代码，此处先忽略它。



将上面宏展开以后，就变成：

```
extern "C" { 
  void jni_SetStaticObjectField(JNIEnv *env, jclass clazz, jfieldID fieldID, jobject value) {
      
    /**
    *  这部分来自JNI_ENTRY
    */
    JavaThread* thread=JavaThread::thread_from_jni_environment(env);
    assert( !VerifyJNIEnvThread || (thread == Thread::current()), "JNIEnv is only valid in same thread");
    ThreadInVMfromNative __tiv(thread);
    debug_only(VMNativeEntryWrapper __vew;) 
    VM_ENTRY_BASE(result_type, header, thread)
      
    WeakPreserveExceptionMark __wem(thread);
      
    /**
    *  这部分是SetStaticObjectField的逻辑
    */
    JNIid* id = jfieldIDWorkaround::from_static_jfieldID(fieldID);
    if (JvmtiExport::should_post_field_modification()) {
        jvalue field_value;
        field_value.l = value;
        JvmtiExport::jni_SetField_probe(thread, NULL, NULL, id->holder(), fieldID, true, 'L', (jvalue *)&field_value);
    }
    id->holder()->java_mirror()->obj_field_put(id->offset(), JNIHandles::resolve(value));
    
  } }  // 来自JNI_END
```

现在解释一下JNI_ENTRY里面代码的逻辑：
1. 从env里面获取线程 JavaThread 指针对象
2. 判断当前线程是否JavaThread
3. 创建 ThreadInVMfromNative 对象，用于将JavaThread对象的状态改变（从_thread_in_native改为_thread_in_vm）
4. 创建 HandleMarkCleaner 对象
5. 校验栈对齐
6. 创建 WeakPreserveExceptionMark 对象（这个的作用没明白，以后再看）


以上这些步骤都是一些事务性的工作，所以HotSpot为了维护和扩展，将这些代码提取出来由 JNI_ENTRY 代替。







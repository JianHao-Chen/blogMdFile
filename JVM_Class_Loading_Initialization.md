---
layout: post
title: "HotSpot的类加载--Initialization"
date: 2019-06-11 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---


在学习HotSpot的类加载部分源码的过程中，写下以下几篇文章：
* HotSpot的类加载--概述
* HotSpot的类加载--Loading
* HotSpot的类加载--Linking
* HotSpot的类加载--Initialization（本文）

<!--more-->


### 初始化
类或接口的初始化就是执行它的初始化方法（即类构造器`<clinit>()`方法）。

#### 关于`<clinit>`
* 是由编译器自动收集类中所有类变量的赋值动作和静态语句块合并生成的。
* 虚拟机会保证子类的`<clinit>`执行之前，父类的`<clinit>`已经执行完毕。


#### 实现
看到`instanceKlass::initialize_impl`方法，省略部分代码：
```
void instanceKlass::initialize_impl(instanceKlassHandle this_oop, TRAPS) {

  // Make sure klass is linked (verified) before initialization
  // A class could already be verified, since it has been reflected upon.
  this_oop->link_class(CHECK);

  bool wait = false;

  // 【 Step 1】 获取同步锁
  { ObjectLocker ol(this_oop, THREAD);

    Thread *self = THREAD; // it's passed the current thread

    // 【 Step 2】
    // If we were to use wait() instead of waitInterruptibly() then
    // we might end up throwing IE from link/symbol resolution sites
    // that aren't expected to throw.  This would wreak havoc.  See 6320309.
    // 如果类正在被其他线程初始化，那么释放锁，并且挂起当前线程，直到初始化完成。
    while(this_oop->is_being_initialized() && !this_oop->is_reentrant_initialization(self)) {
        wait = true;
      ol.waitUninterruptibly(CHECK);
    }

    // 【 Step 3】
    // 如果类被当前线程初始化中，那么这个属于递归的调用了初始化方法，只需要释放锁并退出就可以了。
    if (this_oop->is_being_initialized() && this_oop->is_reentrant_initialization(self)) {
      DTRACE_CLASSINIT_PROBE_WAIT(recursive, instanceKlass::cast(this_oop()), -1,wait);
      return;
    }

    // 【 Step 4】
    // 如果类已经初始化完成，那么只需要释放锁并退出就可以了。
    if (this_oop->is_initialized()) {
      DTRACE_CLASSINIT_PROBE_WAIT(concurrent, instanceKlass::cast(this_oop()), -1,wait);
      return;
    }

    // 【 Step 5】
    // 如果类初始化出错，那么释放锁并抛出 NoClassDefFoundError
    if (this_oop->is_in_error_state()) {
      DTRACE_CLASSINIT_PROBE_WAIT(erroneous, instanceKlass::cast(this_oop()), -1,wait);
      ResourceMark rm(THREAD);
      const char* desc = "Could not initialize class ";
      const char* className = this_oop->external_name();
      size_t msglen = strlen(desc) + strlen(className) + 1;
      char* message = NEW_RESOURCE_ARRAY(char, msglen);
      if (NULL == message) {
        // Out of memory: can't create detailed error message
        THROW_MSG(vmSymbols::java_lang_NoClassDefFoundError(), className);
      } else {
        jio_snprintf(message, msglen, "%s%s", desc, className);
        THROW_MSG(vmSymbols::java_lang_NoClassDefFoundError(), message);
      }
    }

    // 【 Step 6】
    // 这种情况表面：这个类就是被当前线程初始化，设置 oop 的状态
    this_oop->set_init_state(being_initialized);
    this_oop->set_init_thread(self);
  }

  // 【 Step 7】
  // 如果当前类不是接口，并且它的父类没有初始化，那么递归调用初始化方法（如果需要的话，进行相应的验证）。
  // 如果父类的初始化失败了：
  // (1) 获取同步锁        (2) 设置当前类状态为初始化失败
  // (3) 通知等待的线程    (4)释放锁。     (5) 抛出错误
  klassOop super_klass = this_oop->super();
  if (super_klass != NULL && !this_oop->is_interface() && Klass::cast(super_klass)->should_be_initialized()) {
    Klass::cast(super_klass)->initialize(THREAD);

    if (HAS_PENDING_EXCEPTION) {
      Handle e(THREAD, PENDING_EXCEPTION);
      CLEAR_PENDING_EXCEPTION;
      {
        EXCEPTION_MARK;
        this_oop->set_initialization_state_and_notify(initialization_error, THREAD); // Locks object, set state, and notify all waiting threads
        CLEAR_PENDING_EXCEPTION;   // ignore any exception thrown, superclass initialization error is thrown below
      }
      DTRACE_CLASSINIT_PROBE_WAIT(super__failed, instanceKlass::cast(this_oop()), -1,wait);
      THROW_OOP(e());
    }
  }

  // 【 Step 8】
  // 调用类构造器
  {
    assert(THREAD->is_Java_thread(), "non-JavaThread in initialize_impl");
    JavaThread* jt = (JavaThread*)THREAD;
    DTRACE_CLASSINIT_PROBE_WAIT(clinit, instanceKlass::cast(this_oop()), -1,wait);
    // Timer includes any side effects of class initialization (resolution,
    // etc), but not recursive entry into call_class_initializer().
    PerfClassTraceTime timer(ClassLoader::perf_class_init_time(),
                             ClassLoader::perf_class_init_selftime(),
                             ClassLoader::perf_classes_inited(),
                             jt->get_thread_stat()->perf_recursion_counts_addr(),
                             jt->get_thread_stat()->perf_timers_addr(),
                             PerfClassTraceTime::CLASS_CLINIT);
    this_oop->call_class_initializer(THREAD);
  }

  // Step 9
  // 如果类构造器执行完成，就：
  // (1) 获取同步锁        (2) 设置当前类状态为初始化完成
  // (3) 通知等待的线程    (4)释放锁。 
  if (!HAS_PENDING_EXCEPTION) {
    this_oop->set_initialization_state_and_notify(fully_initialized, CHECK);
    { ResourceMark rm(THREAD);
      debug_only(this_oop->vtable()->verify(tty, true);)
    }
  }
  else {
    // Step 10 and 11
    Handle e(THREAD, PENDING_EXCEPTION);
    CLEAR_PENDING_EXCEPTION;
    {
      EXCEPTION_MARK;
      this_oop->set_initialization_state_and_notify(initialization_error, THREAD);
      CLEAR_PENDING_EXCEPTION;   // ignore any exception thrown, class initialization error is thrown below
    }
    DTRACE_CLASSINIT_PROBE_WAIT(error, instanceKlass::cast(this_oop()), -1,wait);
    if (e->is_a(SystemDictionary::Error_klass())) {
      THROW_OOP(e());
    } else {
      JavaCallArguments args(e);
      THROW_ARG(vmSymbols::java_lang_ExceptionInInitializerError(),
                vmSymbols::throwable_void_signature(),
                &args);
    }
  }
  DTRACE_CLASSINIT_PROBE_WAIT(end, instanceKlass::cast(this_oop()), -1,wait);
}
```




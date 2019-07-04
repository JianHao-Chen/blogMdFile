---
layout: post
title: "HotSpot是如何实现Java线程的"
date: 2019-06-02 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

Java线程的实现，可以分为如下3层：

![](/assets/blogImg/JVM/Thread/1.png)

下面，将分析HotSpot的实现细节。

<!--more-->

### 线程层次结构
Java线程实现的分层如上图。

#### Java应用层
这一层是我们使用最多的一层（Java 的`Thread`类）。<font color=orange>通过本地方法进入JVM层</font>（调用本地方法（`start0`），在JVM创建相应的实例）。

#### JVM层
用户自定义的线程，就是在JVM层的`JavaThread`。

##### JVM的线程
JVM中有多种线程，不同类型的线程承担着不同的职责。其继承结构如下图：

![](/assets/blogImg/JVM/Thread/2.png)

这些线程大概可以分成两类：
* 虚拟机工作线程：这一类的线程承担了JVM自身的一些工作职责，如监控、GC；
* 用户自定义线程：实际上也就是 JavaThread，对应于用户创建的Thread实例。这些线程主要是执行用户任务；


##### JavaThread的继承关系
JavaThread的继承关系如图：

![](/assets/blogImg/JVM/Thread/3.png)

<font size=4 color=red>注意</font>：
JavaThread是CHeapObj的子类。CHeapObj，顾名思义，是C-heap对象，它被C当中的free和malloc两个操作进行管理。这意味着，JavaThread的实例，并不是在JVM中的堆中被管理的，也就是，它<font color=red>不会被垃圾回收所管理</font>。在后面我们将看到，在执行了线程的entry_point之后，线程会被显示调用的delete所销毁。

##### JavaThread的创建
JavaThread的创建，是通过调用glibc库的pthread_create方法的。具体可以看`os::create_thread`方法。

pthread_create方法的定义：
```
// glibc/nptl/nptl/sysdeps/pthread/pthread.h
extern int pthread_create (pthread_t *__restrict __newthread,
            const pthread_attr_t *__restrict __attr,
            void *(*__start_routine) (void *),
            void *__restrict __arg) __THROWNL __nonnull ((1, 3));
```
该方法的参数依次是：
* __newthread: 新创建的线程ID会被设置到该参数指向的内存单元
* __attr: 该参数用于定制不同的线程属性
* __start_routine: 这是一个函数指针，新创建的线程将从函数开始的地址开始运行
* __arg: 这是一个无类型指针参数，是__start_routine函数的唯一的参数



##### JavaThread的运行
JavaThread创建后，会调用`java_start`方法，它定义在 ox_linux.cpp，作为启动例程（start routine）在调用`pthread_create`时指定。

这个方法在进行以下设置后，会调用`JavaThread::run`方法：
```
void JavaThread::run() {
  // initialize thread-local alloc buffer related fields
  // 准备 tlab
  this->initialize_tlab();

  // Record real stack base and size.
  // 栈基址、大小
  this->record_stack_base_and_size();

  // Initialize thread local storage; set before calling MutexLocker
  this->initialize_thread_local_storage();

  this->create_stack_guard_pages();

  this->cache_global_variables();

  // Thread is now sufficient initialized to be handled by the safepoint code as being
  // in the VM. Change thread state from _thread_new to _thread_in_vm
  // 设置线程状态，这样才能被JVM使用
  ThreadStateTransition::transition_and_fence(this, _thread_new, _thread_in_vm);

  
  // 触发一个线程开始的事件
  EventThreadStart event;
  if (event.should_commit()) {
     event.set_javalangthread(java_lang_Thread::thread_id(this->threadObj()));
     event.commit();
  }

  // 这里调用 entry_point，进入应用程序的逻辑
  thread_main_inner();

  // Note, thread is no longer valid at this point!
  // 到这里，表示线程已经完成使命，已经不能再用了。
}
```

其中，令线程执行应用程序逻辑的`thread_main_inner`方法，其实只是调用JavaThread的`_entry_point`，它是函数指针，对应的函数是：
```
static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread->threadObj());
  JavaValue result(T_VOID);
  JavaCalls::call_virtual(&result,
                          obj,
                          KlassHandle(THREAD, SystemDictionary::Thread_klass()),
                          vmSymbols::run_method_name(),
                          vmSymbols::void_method_signature(),
                          THREAD);
}
```
关于`JavaCalls`以后再说，上面代码做的是调用Java线程（用户定义的那个）的`run`方法。


#### 内核层
现在我们将脱离JVM的源码，要进入到glibc和操作系统底层，揭开Java线程实现的最后一道面纱。为了搞清楚这个问题，首先要了解一点
Linux内核中关于线程和进程的基础知识。

* 在Linux系统中，线程又被称为轻量级线程(lightweight process)
* 在Linux系统上创建线程相对还是比较廉价的--它允许调用方决定创建的新的线程共享老线程的哪些信息
* Linux的线程模型是一对一的，就意味着，每一个用户级线程都有一个对应的内核线程。如果我们将JVM创建的线程看成是用户级线程，那么这个线程背后，就是一个内核线程在支持。这种模型对Java线程有很大影响，例如：
  * 当一个线程被阻塞了， 内核可以调度另外一个线程继续执行。
  * Linux系统对进程的限制，大部分都会直接影响到Java线程，比如说线程数量的限制。

### 线程栈空间分配
* <font color=red>JVM没有显式的栈区，它的栈依赖于系统实现</font>。
* <font color=red>JVM对线程栈的管理，都是通过glibc的调用来实现的，归根到底是操作系统在管理线程栈</font>。

分配栈空间的代码在pthread_create.c：
```
int err = ALLOCATE_STACK (iattr, &pd);
if (__builtin_expect (err != 0, 0))
  /* Something went wrong.  Maybe a parameter of the attributes is
     invalid or we could not allocate memory.  Note we have to
     translate error codes.  */
  return err == ENOMEM ? EAGAIN : err;
```
ALLOCATE_STACK是一个宏，它只是调用了:
```
// glibc/nptl/allocatestack.c
static int
allocate_stack (const struct pthread_attr *attr, struct pthread **pdp,
        ALLOCATE_STACK_PARMS)
```
该方法使用到了一个我们在JVM中时常配置的参数-Xss，线程栈的大小(通过将栈大小设置到pthread_attr)。


<font size=3>**影响栈大小的设置**</font>
* <font color=red>栈内分配</font>：对于一个对象来说，如果经过逃逸分析，可以确定这个对象只会在方法内部被使用，那么这个对象可以直接被分配在栈上，而不必分配到堆上。这种优化会极大的影响栈大小的设置。因为相比仅仅存放一个指针而言，存放一个完整的对象，所需的空间是要多很多的;
* <font color=red>栈帧重叠</font>：在HotSpot内部，每一次方法调用，调用者的操作数栈和被调用者的局部变量表是有一部分重叠，这能够节省一部分的空间;


### JVM对线程栈空间的管理
虽然HotSpot的栈是依赖于系统实现，JVM还是需要获取栈的基址、大小，判断是否会访问栈以外的地址（即栈溢出）。

#### JVM的栈空间
以Linux为例，看到os_linux_x86.cpp的注释：
```
// Java thread:
//
//   Low memory addresses
//    +------------------------+
//    |                        |\  JavaThread created by VM does not have glibc
//    |    glibc guard page    | - guard, attached Java thread usually has
//    |                        |/  1 page glibc guard.
// P1 +------------------------+ Thread::stack_base() - Thread::stack_size()
//    |                        |\
//    |  HotSpot Guard Pages   | - red and yellow pages
//    |                        |/
//    +------------------------+ JavaThread::stack_yellow_zone_base()
//    |                        |\
//    |      Normal Stack      | -
//    |                        |/
// P2 +------------------------+ Thread::stack_base()
//
// Non-Java thread:
//
//   Low memory addresses
//    +------------------------+
//    |                        |\
//    |  glibc guard page      | - usually 1 page
//    |                        |/
// P1 +------------------------+ Thread::stack_base() - Thread::stack_size()
//    |                        |\
//    |      Normal Stack      | -
//    |                        |/
// P2 +------------------------+ Thread::stack_base()
//
// ** P1 (aka bottom) and size ( P2 = P1 - size) are the address and stack size returned from
//    pthread_attr_getstack()
```
总结：
* Linux下线程栈是往低地址方式增长的（栈顶在低地址）
* 对于Java线程，存在一块叫“HotSpot Guard Pages”的区域
 
#### HotSpot的Guard Pages

##### Guard Pages机制
HotSpot Guard Pages 包含 <font color=red>**red page**</font> 和 <font color=cream>**yellow page**</font>：
* <font color=red>**red page**</font>：在 yellow page 的“上面”。
* <font color=cream>**yellow page**</font>：是可恢复栈溢出处理用的。
* <font color=red>**red page**</font> 是不可恢复栈溢出用的。

HotSpot VM的实现有个特别的地方，就是栈溢出的处理是在发生溢出的线程上进行的，而且继续使用该线程原本的栈（而不使用alternative stack）。这就意味着某个线程在本来已经发生栈溢出的状况下要处理栈溢出异常还得进一步使用更多栈空间，这是个悖论—-***已经没栈空间用了***。

所以预留出来的yellow page在平时处于不可读写状态，一旦被读写就意味着发生了栈溢出。在处理栈溢出时会临时把<font color=cream>**yellow page**</font>的权限改为可读写，“借给”处理函数分配其栈桢，完事之后再恢复为不可读写状态。

而如果访问到了<font color=red>**red page**</font>那就彻底超过极限了，HotSpot VM就会让进程直接crash掉。

##### Guard Pages创建
以Linux平台为例。

在`JavaThread::run`方法里面，会调用`JavaThread::create_stack_guard_pages`方法：
```
  if (! os::uses_stack_guard_pages() || _stack_guard_state != stack_guard_unused) return;
  
  address low_addr = stack_base() - stack_size();
  size_t len = (StackYellowPages + StackRedPages) * os::vm_page_size();

  int allocate = os::allocate_stack_guard_pages();
  // warning("Guarding at " PTR_FORMAT " for len " SIZE_FORMAT "\n", low_addr, len);

  if (allocate && !os::create_stack_guard_pages((char *) low_addr, len)) {
    warning("Attempt to allocate stack guard pages failed.");
    return;
  }

  if (os::guard_memory((char *) low_addr, len)) {
    _stack_guard_state = stack_guard_enabled;
  } else {
    warning("Attempt to protect stack guard pages failed.");
    if (os::uncommit_memory((char *) low_addr, len)) {
      warning("Attempt to deallocate stack guard pages failed.");
    }
  }
```
* 通过`os::create_stack_guard_pages`方法创建 guard page，最后调用的是系统调用`mmap`
* 通过`os::guard_memory`方法对guard page进行保护
  * 实际上通过系统调用`mprotect`，将guard page设置为不允许进程访问
  * 当有进程访问了，内核会创建一个 ***SIGSEGV*** 信号给这个进程。
  

##### 对SIGSEGV 信号的处理
在 os_linux_x86.cpp 文件下的 `JVM_handle_linux_signal`方法：
```
// Handle ALL stack overflow variations here
if (sig == SIGSEGV) {
  address addr = (address) info->si_addr;
  
  // check if fault address is within thread stack
  if (addr < thread->stack_base() &&
      addr >= thread->stack_base() - thread->stack_size()) {
      
    // stack overflow
    if (thread->in_stack_yellow_zone(addr)) {
      thread->disable_stack_yellow_zone();
      if (thread->thread_state() == _thread_in_Java) {
        // Throw a stack overflow exception.  Guard pages will bereenabled
        // while unwinding the stack.
        stub = SharedRuntime::continuation_for_implicit_exception(thred, pc, SharedRuntime::STACK_OVERFLOW);
      } else {
        // Thread was in the vm or native code.  Return and try to finish.
        return 1;
      }
    } 
    else if (thread->in_stack_red_zone(addr)) {
      // Fatal red zone violation.  Disable the guard pages and fallthrough
      // to handle_unexpected_exception way down below.
      thread->disable_stack_red_zone();
      tty->print_raw_cr("An irrecoverable stack overflow hasoccurred.");
    }
    else {
      // Accessing stack address below sp may cause SEGV if current
      // thread has MAP_GROWSDOWN stack. This should only happen when
      // current thread was created by user code with MAP_GROWSDOWNflag
      // and then attached to VM. See notes in os_linux.cpp.
      if (thread->osthread()->expanding_stack() == 0) {
         thread->osthread()->set_expanding_stack();
         if (os::Linux::manually_expand_stack(thread, addr)) {
           thread->osthread()->clear_expanding_stack();
           return 1;
         }
         thread->osthread()->clear_expanding_stack();
      } else {
         fatal("recursive segv. expanding stack.");
      }
    }
  }
}
```
获取产生 SIGSEGV 信号的地址：
* 如果处于 <font color=cream>**yellow page**</font> 区域，那么产生 stack overflow exception。
* 如果处于 <font color=red>**red page**</font> 区域，将导致JVM进程退出，不过会有 crash 日志。

另外，如果处于 red page 区域，并且没有栈空间了，这时连这个信号处理函数都进不去，进程会crash并且没有日志产生。

---
layout: post
title: "HotSpot的SafePoint"
date: 2019-06-2308:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文内容大纲：
* 什么是 SafePoint
* 线程是如何进入 SafePoint
* 线程是如何从 SafePoint 恢复

<!--more-->

### 什么是 SafePoint ？
<font color=orange>**SafePoint（安全点）其实是 HotSpot 的 Stop the world 机制**。</font>

JVM采用了主动式阻塞的方式，Java线程不是随时都可以进入阻塞，需要运行到特定的点，叫safepoint，在这些点的位置Java线程可以被全部阻塞，整个堆的状态是一个暂时稳定的状态。

### 线程是如何进入 SafePoint 的（被挂起）？
VMThread 调用`SafepointSynchronize::begin()`

#### 准备工作
```
// Roll all threads forward to a safepoint and suspend them all
void SafepointSynchronize::begin() {

  Thread* myThread = Thread::current();
  assert(myThread->is_VM_thread(), "Only VM thread may execute a safepoint");
  
  // By getting the Threads_lock, we assure that no threads are about to start or
  // exit. It is released again in SafepointSynchronize::end().
  Threads_lock->lock();
  
  int nof_threads = Threads::number_of_threads();
  
  {
  MutexLocker mu(Safepoint_lock);
  
  // Set number of threads to wait for, before we initiate the callbacks
  _waiting_to_block = nof_threads;
  TryingToBlock     = 0 ;
  int still_running = nof_threads;
```
这里做了一些准备工作：
* 验证当前线程是否 VM 线程
* 获取 Safepoint_lock
* 获取需要等待进入 SafePoint 的线程数

#### 不同的线程挂起逻辑
Java thread 可以有多种不同的状态，所以挂起的机制也不同：
```
  // Begin the process of bringing the system to a safepoint.
  // Java threads can be in several different states and are
  // stopped by different mechanisms:
  //
  //  1. Running interpreted
  //     The interpeter dispatch table is changed to force it to
  //     check for a safepoint condition between bytecodes.
  //  2. Running in native code
  //     When returning from the native code, a Java thread must check
  //     the safepoint _state to see if we must block.  If the
  //     VM thread sees a Java thread in native, it does
  //     not wait for this thread to block.  The order of the memory
  //     writes and reads of both the safepoint state and the Java
  //     threads state is critical.  In order to guarantee that the
  //     memory writes are serialized with respect to each other,
  //     the VM thread issues a memory barrier instruction
  //     (on MP systems).  In order to avoid the overhead of issuing
  //     a memory barrier for each Java thread making native calls, each Java
  //     thread performs a write to a single memory page after changing
  //     the thread state.  The VM thread performs a sequence of
  //     mprotect OS calls which forces all previous writes from all
  //     Java threads to be serialized.  This is done in the
  //     os::serialize_thread_states() call.  This has proven to be
  //     much more efficient than executing a membar instruction
  //     on every call to native code.
  //  3. Running compiled Code
  //     Compiled code reads a global (Safepoint Polling) page that
  //     is set to fault if we are trying to get to a safepoint.
  //  4. Blocked
  //     A thread which is blocked will not be allowed to return from the
  //     block condition until the safepoint operation is complete.
  //  5. In VM or Transitioning between states
  //     If a Java thread is currently running in the VM or transitioning
  //     between states, the safepointing code will wait for the thread to
  //     block itself when it attempts transitions to a new state.
```
根据 Java Thread的运行状态，挂起逻辑可以分为如下5种：
* 在解释模式下：通过解释器的分发表，让线程检查 SafePoint 状态，从而阻塞线程。
* 运行 notive code：
  * VM Thread不等待这个线程，因为这个线程从 native code 返回时，会检查 SafePoint 状态，如果有GC线程执行GC任务，那么这个线程需要等待GC完成。
  * 需要线程状态同步。
* 运行 compile code：通过设置polling page不可读，当Java thread发现该内存页不可读时，最终会被阻塞挂起。
* 处于 Blocked 状态：
  * JVM引入了 safe region 的概念，就是为了应对处于 Sleep、Blocked状态的线程
  * 线程处于Blocked状态，即使它的条件满足了，也不能马上返回，必须检查直到 safepoint operation完成了。
* 正在转换状态：会去检查safepoint状态，如果需要阻塞，就把自己挂起。

#### 解释模式下，线程进入 SafePoint
在`SafepointSynchronize::begin()`方法里面，会调用`Interpreter::notice_safepoints()`方法：
```
void TemplateInterpreter::notice_safepoints() {
  if (!_notice_safepoints) {
    // switch to safepoint dispatch table
    _notice_safepoints = true;
    copy_table((address*)&_safept_table, (address*)&_active_table, sizeof(_active_table) / sizeof(address));
  }
}
```
`TemplateInterpreter`始终使用 _active_table，而 _safept_table 用于 SafePoint。


在 _safept_table 这个转发表里面，为每个字节码都设置安全点例程（<font color=red>**这里待确认!**</font>）：
```
void TemplateInterpreterGenerator::set_safepoints_for_all_bytes() {
  for (int i = 0; i < DispatchTable::length; i++) {
    Bytecodes::Code code = (Bytecodes::Code)i;
    if (Bytecodes::is_defined(code)) Interpreter::_safept_table.set_entry(code, Interpreter::_safept_entry);
  }
}
```
而这个`_safept_entry`的创建在`TemplateInterpreterGenerator::generate_all`里面：
```
  Interpreter::_safept_entry =
      EntryPoint(
        generate_safept_entry_for(btos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(ctos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(stos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(atos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(itos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(ltos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(ftos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(dtos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint)),
        generate_safept_entry_for(vtos, CAST_FROM_FN_PTR(address, InterpreterRuntime::at_safepoint))
      );
```
继续看`InterpreterRuntime::at_safepoint`定义的地方：
```
IRT_ENTRY(void, InterpreterRuntime::at_safepoint(JavaThread* thread))
  // ... 省略
IRT_END
```
这里，通过宏定义`IRT_ENTRY`，会声明一个`ThreadInVMfromJava`变量，它的析构函数会调用`transition`方法，里面包含了使线程阻塞的逻辑：
```
static inline void transition(JavaThread *thread, JavaThreadState from, JavaThreadState to) {
    assert(from != _thread_in_Java, "use transition_from_java");
    assert(from != _thread_in_native, "use transition_from_native");
    assert((from & 1) == 0 && (to & 1) == 0, "odd numbers are transitions states");
    assert(thread->thread_state() == from, "coming from wrong thread state");
    // Change to transition state (assumes total store ordering!  -Urs)
    thread->set_thread_state((JavaThreadState)(from + 1));

    // Make sure new state is seen by VM thread
    if (os::is_MP()) {
      if (UseMembar) {
        // Force a fence between the write above and read below
        OrderAccess::fence();
      } else {
        // store to serialize page so VM thread can do pseudo remote membar
        os::write_memory_serialize_page(thread);
      }
    }

    if (SafepointSynchronize::do_call_back()) {
      SafepointSynchronize::block(thread);
    }
    thread->set_thread_state(to);

    CHECK_UNHANDLED_OOPS_ONLY(thread->clear_unhandled_oops();)
}
```
上面代码有2个重要的点：
* 线程同步相关的
* 使用`SafepointSynchronize::block`使当前线程阻塞


#### 状态同步

前面提到过，当线程状态改变时，Java Thread会执行如下代码将线程改变的状态同步出去：
```
  // Change to transition state (assumes total store ordering!  -Urs)
  thread->set_thread_state((JavaThreadState)(from + 1));

  // Make sure new state is seen by VM thread
  if (os::is_MP()) {
    if (UseMembar) {
      // Force a fence between the write above and read below
      OrderAccess::fence();
    } else {
      // store to serialize page so VM thread can do pseudo remote membar
      os::write_memory_serialize_page(thread);
    }
  }
  
  
  static inline void write_memory_serialize_page(JavaThread *thread) {
    uintptr_t page_offset = ((uintptr_t)thread >>
                            get_serialize_page_shift_count()) &
                            get_serialize_page_mask();
    *(volatile int32_t *)((uintptr_t)_mem_serialize_page+page_offset) = 1;
  }
```


<font color=purple>内存屏障</font>是一类同步屏障指令，一项在多CPU下保证内存访问顺序的技术，但这是昂贵的操作。

HotSpot为了提升性能，采用 Serialization Page 的方式“模拟”内存屏障。

<font color=red>Serialization Page</font>：
* JVM启动时，会初始化并设置内存页`_mem_serialize_page`给其他的线程写。
* 通过算法保证，不同的线程，写到这个page的不同位置。
* Java 线程通过调用`write_memory_serialize_page`方法，向page写入数值1。
* VMThread通过调用`os::serialize_thread_states()`方法，获得Java线程状态的改变，具体如下：
  * `serialize_thread_states`方法：
  ```
  // Serialize all thread state variables
    void os::serialize_thread_states() {
    // On some platforms such as Solaris & Linux, the time duration of the   page
    // permission restoration is observed to be much longer than expected    due to
    // scheduler starvation problem etc. To avoid the long synchronization
    // time and expensive page trap spinning, 'SerializePageLock' is used to   block
    // the mutator thread if such case is encountered. See bug 6546278 for   details.
    Thread::muxAcquire(&SerializePageLock, "serialize_thread_states");
    os::protect_memory((char *)os::get_memory_serialize_page(),
                       os::vm_page_size(), MEM_PROT_READ);
    os::protect_memory((char *)os::get_memory_serialize_page(),
                       os::vm_page_size(), MEM_PROT_RW);
    Thread::muxRelease(&SerializePageLock);
  }
  ```
  * 通过系统调用`protect_memory`，将page设置为可读，由`protect_memory`<font color=red>保证前面对线程状态的写发生在这个保护页的写之前，即保证之前的对java线程状态的修改操作为其他所有线程都可见</font>。
  * 再次将page设置为可读写，等待后续的`write_memory_serialize_page`操作
* 使用 Serialization Page 比 Memory Barrier的性能好：
  * 因为Java线程只做 Serialization Page的写，不需要Memory Barrier操作
  * VMThread需要做 mprotect + memory barrier操作（会更慢）
  * 但是 VMThread 只有一个， Java Thread有多个，实际上是赚了。
  

#### 运行编译代码的线程进入 SafePoint
* <font color=lilac>Poling page</font>：在jvm初始化启动的时候会初始化的一个单独的内存页面，这个页面是让运行的编译过的代码的线程进入停止状态的关键。
* java 的JIT 会直接编译一些热门的源码到机器码，直接执行而不需要在解释执行从而提高效率，在编译的代码中，当函数或者方法块返回的时候会去访问一个内存poling页面。
* 当进入 SafePoint 时，VMThread会调用`os::make_polling_page_unreadable`将设置polling page不可读。
* 在Linux下，`make_polling_page_unreadable`实际上使用系统调用`mprotect`。
* 当Java线程（运行着编译代码），访问到这个polling page时，会得到 SIGSEGV 信号。
* JVM对信号 SIGSEGV 的处理（在 os_linux_x86.cpp）：
  ```
    // Check to see if we caught the safepoint code in the
    // process of write protecting the memory serialization page.
    // It write enables the page immediately after protecting it
    // so we can just return to retry the write.
    if ((sig == SIGSEGV) &&
        os::is_memory_serialize_page(thread, (address) info->si_addr)) {
      // Block current thread until the memory serialize page permission restored.
      os::block_on_serialize_page_trap();
      return true;
    }
  ```
 * 通过`os::block_on_serialize_page_trap`令当前线程阻塞。
  

####  SafeRegion
* safepoint只能处理正在运行的线程，它们可以主动运行到safepoint。而一些Sleep或者被blocked的线程不能主动运行到safepoint。
* 因此JVM引入了safe region的概念：
  * safe region是指一块区域，这块区域中的引用都不会被修改，比如线程被阻塞了，那么它的线程堆栈中的引用是不会被修改的，JVM可以安全地进行标记。
  * 线程进入到safe region的时候先标识自己进入了safe region，等它被唤醒准备离开safe region的时候，先检查能否离开，如果GC已经完成，那么可以离开，否则就在safe region呆着。


### 线程是如何从 SafePoint 恢复的？
通过VMThread执行`SafepointSynchronize::end()`方法：
```
// Wake up all threads, so they are ready to resume execution after the safepoint
// operation has been carried out
void SafepointSynchronize::end() {
```

#### 设置polling page可读
```
  if (PageArmed) {
    // Make polling safepoint aware
    os::make_polling_page_readable();
    PageArmed = 0 ;
  }
```

#### 恢复解释器的分发表
```
// switch from the dispatch table which notices safepoints back to the
// normal dispatch table.  So that we can notice single stepping points,
// keep the safepoint dispatch table if we are single stepping in JVMTI.
// Note that the should_post_single_step test is exactly as fast as the
// JvmtiExport::_enabled test and covers both cases.
void TemplateInterpreter::ignore_safepoints() {
  if (_notice_safepoints) {
    if (!JvmtiExport::should_post_single_step()) {
      // switch to normal dispatch table
      _notice_safepoints = false;
      copy_table((address*)&_normal_table, (address*)&_active_table, sizeof(_active_table) / sizeof(address));
    }
  }
}
```
将分发表`_active_table`恢复为原来的`_normal_table`。

#### 唤醒等待的线程
```
// Start suspended threads
for(JavaThread *current = Threads::first(); current; current =current->next()) {
  // A problem occurring on Solaris is when attempting to restartthreads
  // the first #cpus - 1 go well, but then the VMThread is preemptedwhen we get
  // to the next one (since it has been running the longest).  We thenhave
  // to wait for a cpu to become available before we can continuerestarting
  // threads.
  // FIXME: This causes the performance of the VM to degrade whenactive and with
  // large numbers of threads.  Apparently this is due to thesynchronous nature
  // of suspending threads.
  //
  // TODO-FIXME: the comments above are vestigial and no longer apply.
  // Furthermore, using solaris' schedctl in this particular contextconfers no benefit
  if (VMThreadHintNoPreempt) {
    os::hint_no_preempt();
  }
  ThreadSafepointState* cur_state = current->safepoint_state();
  assert(cur_state->type() != ThreadSafepointState::_running, "Threadnot suspended at safepoint");
  cur_state->restart();
  assert(cur_state->is_running(), "safepoint state has not beenreset");
}
```

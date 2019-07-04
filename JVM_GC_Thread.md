---
layout: post
title: "HotSpot里面的GC线程(VMThread)"
date: 2019-06-20 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

HotSpot里面并没有“GCThread”这样的对象，GC操作是由 **VMThread** 负责的。

<!--more-->


### VMThread
* 是一个单例对象
* 它会产生或触发其他所有的线程
* 它被其他线程用来做一些VM操作，例如GC

####  VMThread的初始化
在JVM初始化时，会创建VMThread,具体看到` Threads::create_vm`方法：
```
  // Create the VMThread
  { TraceTime timer("Start VMThread", TraceStartupTime);
    VMThread::create();
    Thread* vmthread = VMThread::vm_thread();

    if (!os::create_thread(vmthread, os::vm_thread))
      vm_exit_during_initialization("Cannot create VM thread. Out of system resources.");

    // Wait for the VM thread to become ready, and VMThread::run to initialize
    // Monitors can have spurious returns, must always check another state flag
    {
      MutexLocker ml(Notify_lock);
      os::start_thread(vmthread);
      while (vmthread->active_handles() == NULL) {
        Notify_lock->wait();
      }
    }
  }
```

通过调用`VMThread::create()`方法完成`VMThread`的创建：
```
bool              VMThread::_should_terminate   = false;
bool              VMThread::_terminated         = false;
Monitor*          VMThread::_terminate_lock     = NULL;
VMThread*         VMThread::_vm_thread          = NULL;
VM_Operation*     VMThread::_cur_vm_operation   = NULL;
VMOperationQueue* VMThread::_vm_queue           = NULL;
PerfCounter*      VMThread::_perf_accumulated_vm_operation_time = NULL;

void VMThread::create() {
  assert(vm_thread() == NULL, "we can only allocate one VMThread");
  _vm_thread = new VMThread();

  // Create VM operation queue
  _vm_queue = new VMOperationQueue();
  guarantee(_vm_queue != NULL, "just checking");

  _terminate_lock = new Monitor(Mutex::safepoint, "VMThread::_terminate_lock", true);

  if (UsePerfData) {
    // jvmstat performance counters
    Thread* THREAD = Thread::current();
    _perf_accumulated_vm_operation_time =
                 PerfDataManager::create_counter(SUN_THREADS, "vmOperationTime",
                                                 PerfData::U_Ticks, CHECK);
  }
}
```
上面2段代码，`VMThread`的初始化过程可以总结如下：
* `VMThread`包含了如下的成员变量：
  * _should_terminate：与线程同步相关
  * _terminated：表示线程状态
  * _terminate_lock： 互斥锁
  * _vm_thread：VMThread对象，因为是单例
  * _cur_vm_operation：当前的vm操作
  * _vm_queue：队列，用于保存 vm操作
* `VMThread::create()`方法创建 *VMThread* 对象、*VMOperationQueue*、*Monitor*
* VMThread只是一个c++对象，最后还是要与系统线程对应的，通过`os::create_thread`为这个 vmThread 创建系统线程。
* 通过`os::start_thread`启动 VMThread
* 等待 VMThread ready（怎样是Ready？具体看到 ` VMThread::run` 方法）。


#### VMThread的运行
```
void VMThread::run() {

  this->initialize_thread_local_storage();
  this->record_stack_base_and_size();
  
  // Notify_lock wait checks on active_handles() to rewait in
  // case of spurious wakeup, it should wait on the last
  // value set prior to the notify
  this->set_active_handles(JNIHandleBlock::allocate_block());

  {
    MutexLocker ml(Notify_lock);
    Notify_lock->notify();
  }
  
  // Wait for VM_Operations until termination
  this->loop();
  
  // ... 省略
}
```
VMThread主要是做了一些准备工作，通知主线程恢复执行，然后不断轮询处理 VM_Operation。

#### VMThread的轮询
这部分涉及安全点相关的知识（先跳过）并且省略部分代码：
```
void VMThread::loop() {

  while(true) {
    VM_Operation* safepoint_ops = NULL;
    
    //
    // Wait for VM operation
    //
    // use no_safepoint_check to get lock without attempting to "sneak"
    { MutexLockerEx mu_queue(VMOperationQueue_lock,
                             Mutex::_no_safepoint_check_flag);
                             
      // Look for new operation
      assert(_cur_vm_operation == NULL, "no current one should be executing");
      _cur_vm_operation = _vm_queue->remove_next();                         
      
      while (!should_terminate() && _cur_vm_operation == NULL) {
      
        // wait with a timeout to guarantee safepoints at regular intervals
        bool timedout =
          VMOperationQueue_lock->wait(Mutex::_no_safepoint_check_flag,
                                      GuaranteedSafepointInterval);
        
        
        if (timedout && (SafepointALot ||
                         SafepointSynchronize::is_cleanup_needed())) {
          MutexUnlockerEx mul(VMOperationQueue_lock,
                              Mutex::_no_safepoint_check_flag);
          // Force a safepoint since we have not had one for at least
          // 'GuaranteedSafepointInterval' milliseconds.  This will run all
          // the clean-up processing that needs to be done regularly at a
          // safepoint
          SafepointSynchronize::begin();
          #ifdef ASSERT
            if (GCALotAtAllSafepoints) InterfaceSupport::check_gc_alot();
          #endif
          SafepointSynchronize::end();
        }
        _cur_vm_operation = _vm_queue->remove_next();
        
        // If we are at a safepoint we will evaluate all the operations that
        // follow that also require a safepoint
        if (_cur_vm_operation != NULL &&
            _cur_vm_operation->evaluate_at_safepoint()) {
          safepoint_ops = _vm_queue->drain_at_safepoint_priority();
        }
      }
      
       if (should_terminate()) break;
    }  // Release mu_queue_lock
  
  
    //
    // Execute VM operation
    //
    {
        
        // ... 省略
        
        // If we are at a safepoint we will evaluate all the operations that
        // follow that also require a safepoint
        if (_cur_vm_operation->evaluate_at_safepoint()) {
        
          _vm_queue->set_drain_list(safepoint_ops); // ensure ops can be   scanned
          
          SafepointSynchronize::begin();
          evaluate_operation(_cur_vm_operation);
          
          // now process all queued safepoint ops, iteratively draining
          // the queue until there are none left
          do {
            _cur_vm_operation = safepoint_ops;
            if (_cur_vm_operation != NULL) {
              do {
                // evaluate_operation deletes the op object so we have
                // to grab the next op now
                VM_Operation* next = _cur_vm_operation->next();
                _vm_queue->set_drain_list(next);
                evaluate_operation(_cur_vm_operation);
                _cur_vm_operation = next;
                if (PrintSafepointStatistics) {
                  SafepointSynchronize::inc_vmop_coalesced_count();
                }
              } while (_cur_vm_operation != NULL);
            }
            // There is a chance that a thread enqueued a safepoint op
            // since we released the op-queue lock and initiated the   safepoint.
            // So we drain the queue again if there is anything there, as an
            // optimization to try and reduce the number of safepoints.
            // As the safepoint synchronizes us with JavaThreads we will see
            // any enqueue made by a JavaThread, but the peek will not
            // necessarily detect a concurrent enqueue by a GC thread, but
            // that simply means the op will wait for the next major cycle of   the
            // VMThread - just as it would if the GC thread lost the race for
            // the lock.
            if (_vm_queue->peek_at_safepoint_priority()) {
              // must hold lock while draining queue
              MutexLockerEx mu_queue(VMOperationQueue_lock,
                                       Mutex::_no_safepoint_check_flag);
              safepoint_ops = _vm_queue->drain_at_safepoint_priority();
            } else {
              safepoint_ops = NULL;
            }
          } while(safepoint_ops != NULL);
          
          _vm_queue->set_drain_list(NULL);
          
          // Complete safepoint synchronization
          SafepointSynchronize::end();
          
        } else {  // not a safepoint operation
        
          // ... 省略
        
          evaluate_operation(_cur_vm_operation);
          
          _cur_vm_operation = NULL;
        }
    }
    
    //
    //  Notify (potential) waiting Java thread(s) - lock without safepoint
    //  check so that sneaking is not possible
    { MutexLockerEx mu(VMOperationRequest_lock,
                       Mutex::_no_safepoint_check_flag);
      VMOperationRequest_lock->notify_all();
    }
    
    
    //
    // We want to make sure that we get to a safepoint regularly.
    //
    if (SafepointALot || SafepointSynchronize::is_cleanup_needed()) {
      long interval          =    SafepointSynchronize::last_non_safepoint_interval();
      bool max_time_exceeded = GuaranteedSafepointInterval != 0 &&  (interval   > GuaranteedSafepointInterval);
      if (SafepointALot || max_time_exceeded) {
        HandleMark hm(VMThread::vm_thread());
        SafepointSynchronize::begin();
        SafepointSynchronize::end();
      }
    }
  }
}
```
`VMThread`轮询的逻辑如下（不考虑需要 terminate 的情况）：
* 等待 VM Operation：
  *  获取 vm operation queue 的锁
  *  获取队列中的 vm operation
     * 如果为空，在 while 循环里通过 `VMOperationQueue_lock->wait`等待，直到有 vm operation
     * 不为空，进入 **执行 VM Operation** 的流程
* 执行 VM Operation：
  * 通过`VMThread::evaluate_operation`方法执行，具体调用链为：
  `evaluate_operation() ---> VM_Operation::evaluate() --> VM_Operation::doit()`。
  * 各个具体的 `VM_Operation`子类实现自己`doit()`方法的逻辑
* 通知其他Java线程
* 安全点相关（省略）



####  VM Operation的提交

以永久代分配内存失败，需要执行GC为例（看到`PermGen::mem_allocate_in_gen`方法）：

```
VM_GenCollectForPermanentAllocation op(size, 
                                       gc_count_before,
                                       full_gc_count_before,
                                       next_cause);
VMThread::execute(&op);
```

看到`VMThread::execute`方法：
```
void VMThread::execute(VM_Operation* op) {
  Thread* t = Thread::current();
  
  if (!t->is_VM_thread()) {
  
    // Add VM operation to list of waiting threads. We are guaranteed not to block while holding the
    // VMOperationQueue_lock, so we can block without a safepoint check. This allows vm operation requests
    // to be queued up during a safepoint synchronization.
    {
      VMOperationQueue_lock->lock_without_safepoint_check();
      bool ok = _vm_queue->add(op);
      op->set_timestamp(os::javaTimeMillis());
      VMOperationQueue_lock->notify();
      VMOperationQueue_lock->unlock();
      // VM_Operation got skipped
      if (!ok) {
        assert(concurrent, "can only skip concurrent tasks");
        if (op->is_cheap_allocated()) delete op;
        return;
      }
    }
    
    
    if (!concurrent) {
      // Wait for completion of request (non-concurrent)
      // Note: only a JavaThread triggers the safepoint check when locking
      MutexLocker mu(VMOperationRequest_lock);
      while(t->vm_operation_completed_count() < ticket) {
        VMOperationRequest_lock->wait(!t->is_Java_thread());
      }
    }
  
  }  else {
    // invoked by VM thread; usually nested VM operation
    // ... 省略
  }
}
```
关键在于这几个步骤：
* 这里的普通线程获取 VMOperationQueue 的锁
* 将 VMOperation 添加到队列里面
* 调用`notify`唤醒`VMThread`
* 释放锁
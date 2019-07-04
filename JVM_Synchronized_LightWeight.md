---
layout: post
title: "Synchronized底层实现—-轻量级锁"
date: 2019-06-25 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

学习JVM源码Synchronized锁部分，写了如下几篇文章：
* Synchronized底层实现—-概论
* Synchronized底层实现—-偏向锁
* Synchronized底层实现—-轻量级锁（本文）
* Synchronized底层实现—-重量级锁

<!--more-->


### 轻量级锁的获取
代码看到字节码`monitorenter`的实现（在bytecodeInterpreter.cpp， 与偏向锁的入口是一样的），省略部分关于偏向锁的代码:
```
CASE(_monitorenter): {

  //  ... 省略
  
  // traditional lightweight locking
  if (!success) {
  
    // ## 1. 构建一个无锁状态的Displaced Mark Word
    markOop displaced = lockee->mark()->set_unlocked();
    
    // ## 2. 设置到Lock Record
    entry->lock()->set_displaced_header(displaced);
    bool call_vm = UseHeavyMonitors;
    
    // ## 3. 如果CAS替换不成功，代表【锁对象不是无锁状态】，需要判断下是否【锁重入】
    if (call_vm || Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
      
      // ## 4. 如果是【锁重入】
      // （1） 实现原理是：检查锁对象头存储的 Lock Record 地址是否在线程栈中
      // （2） 将这个 Lock Record 的 Displaced Mark Word 设置为 null
      if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
        entry->lock()->set_displaced_header(NULL);
      } 
      // ## 5. 如果不是【锁重入】，说明发生了竞争，需要膨胀为重量级锁
      else {
        CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry)handle_exception);
      }
    }
  }
  UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
  else {
    istate->set_msg(more_monitors);
    UPDATE_PC_AND_RETURN(0); // Re-execute
  }
}
```
如果获取轻量级锁失败，进入`InterpreterRuntime::monitorenter`方法：
```
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorenter(JavaThread* thread, BasicObjectLock* elem))
  // ... 省略
  if (UseBiasedLocking) {
    // Retry fast entry if bias is revoked to avoid unnecessary inflation
    ObjectSynchronizer::fast_enter(h_obj, elem->lock(), true, CHECK);
  } else {
    ObjectSynchronizer::slow_enter(h_obj, elem->lock(), CHECK);
  }
IRT_END  
```
其中，`fast_enter`分支在偏向锁那里讲过了，会先尝试获取偏向锁。如果失败了，执行以下步骤：
* 撤销偏向锁
* 锁升级为轻量级锁（此时，锁的拥有者还是原来的线程）
* 进入`slow_enter`

看到`slow_enter`方法：
```
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");
  
  // ## 1. 如果是【无锁状态】
  if (mark->is_neutral()) {
    lock->set_displaced_header(mark);
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }
    // Fall through to inflate() ...
  } 
  // ## 2. 如果是【锁重入】，设置 Displaced Mark Word 为 null
  else if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }
  
  // ## 3. 存在线程竞争锁，需要膨胀为重量级锁
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
```

【小结】轻量级锁的加锁过程：
* (1) 在线程栈中创建一个 Lock Record
  * 将其`obj`字段指向锁对象
  * 将对象头的 Mark Word 保存在`lock`字段
* (2) 通过 CAS 指令将 Lock Record 的地址存储在对象头的 mark word 中：
  * 如果修改成功（对象处于无锁状态），该线程获得了轻量级锁。
  * 如果失败，进入到步骤3。
* (3) 如果是当前线程已经持有该锁了，代表这是一次锁重入。设置 Lock Record 的 Displaced Mark Word 为 null。然后结束。
* (4) 走到这一步说明发生了竞争，需要膨胀为重量级锁。


### 轻量级锁释放
代码看到字节码`monitorexit`的实现，其实和偏向锁释放是差不多的：
```
CASE(_monitorexit): {
  oop lockee = STACK_OBJECT(-1);
  
  BasicObjectLock* limit = istate->monitor_base();
  BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
  
  // 遍历栈的 Lock Record
  while (most_recent != limit ) {
  
    // ## 1. 找到关联锁对象的 Lock Record
    if ((most_recent)->obj() == lockee) {
      BasicLock* lock = most_recent->lock();
      markOop header = lock->displaced_header();
      // 设置 obj 字段为 null
      most_recent->set_obj(NULL);
      
      // ## 2. 如果是偏向锁，可以返回了，否则走轻量级锁、重量级锁的释放流程。
      if (!lockee->mark()->has_bias_pattern()) {
        bool call_vm = UseHeavyMonitors;
        if (header != NULL || call_vm) {
          if (call_vm || Atomic::cmpxchg_ptr(header, lockee->mark_addr(), lock) != lock) {
            most_recent->set_obj(lockee);
            CALL_VM(InterpreterRuntime::monitorexit(THREAD, most_recent), handle_exception);
          }
        }
      }
      
      UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
    }
    most_recent++;
  }
  // Need to throw illegal monitor state exception
  CALL_VM(InterpreterRuntime::throw_illegal_monitor_state_exception(THREAD), handle_exception);
}
```
主要是看到执行 CAS 操作的那一行：将 Displaced Mark Word 替换到对象头的 mark word 中。如果失败，就需要进入到`InterpreterRuntime::monitorexit`方法中：
```
IRT_ENTRY_NO_ASYNC(void, InterpreterRuntime::monitorexit(JavaThread* thread, BasicObjectLock* elem))

  // ... 省略

  ObjectSynchronizer::slow_exit(h_obj(), elem->lock(), thread);
  elem->set_obj(NULL);
  
  // ... 省略
  
IRT_END  
```
先执行`ObjectSynchronizer::slow_exit`方法，然后释放 Lock Record。


```
void ObjectSynchronizer::slow_exit(oop object, BasicLock* lock, TRAPS) {
  fast_exit (object, lock, THREAD) ;
}


void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {

  markOop dhw = lock->displaced_header();
  markOop mark ;
  
  // ## 1. 如果 Displaced Mark Word == null， 表示之前的获取锁属于【锁重入】
  // 什么都不需要做
  if (dhw == NULL) {
    // ... 省略
  }
  
  mark = object->mark() ;
  
  // ## 2. 如果锁对象的 Mark Word（存的是锁记录地址） == Lock Record 的 Displaced Mark Word：
  //  表示这就是这个线程获得的轻量级锁
  //  只需要使用 CAS 将 Lock Record 中保存的 Displaced Mark Word 替换回对象头的 Mark Word
  if (mark == (markOop) lock) {
     assert (dhw->is_neutral(), "invariant") ;
     if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {
        TEVENT (fast_exit: release stacklock) ;
        return;
     }
  }
  
  // ## 3. 来到这里说明发生了锁竞争：需要膨胀为重量级锁并调用 exit 方法
  ObjectSynchronizer::inflate(THREAD, object)->exit (true, THREAD) ;
}
```




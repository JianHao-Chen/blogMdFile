---
layout: post
title: "Synchronized底层实现—-偏向锁"
date: 2019-06-24 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

学习JVM源码Synchronized锁部分，写了如下几篇文章：
* Synchronized底层实现—-概论
* Synchronized底层实现—-偏向锁（本文）
* Synchronized底层实现—-轻量级锁
* Synchronized底层实现—-重量级锁

<!--more-->


### 前言
偏向锁的实现，其实就是找字节码`monitorenter`的实现，有以下2个地方：
* 在 bytecodeInterpreter：用于c++解释器
* 在 templateInterpreter：用于模板解释器

由于只有少数版本的HotSpot在用 c++ 解释器，其他都用模板解释器，按道理是直接看`templateInterpreter`的实现。只是它们两者的逻辑是差不多的，而`bytecodeInterpreter` 的实现是c++ 代码，而`templateInterpreter`的实现是汇编代码，因此，本文选`bytecodeInterpreter`的实现。


### 偏向锁获取

代码看到`bytecodeInterpreter.cpp`：
```
CASE(_monitorenter): {
  // STACK_OBJECT 就是以（栈顶地址 + 偏移）的方式取出对象
  // 此处，取出的对象就是“锁对象”
  oop lockee = STACK_OBJECT(-1);
  // derefing's lockee ought to provoke implicit null check
  CHECK_NULL(lockee);
  
  // ## 1. 在栈中找到一个 “最近”的 Lock Record（按照从栈顶往下的方向找），有2种情况：
  // (1) Lock Record 是“空闲”的，
  // (2) 已分配给当前对象（即当前属于【锁重入】的情况）
  BasicObjectLock* limit = istate->monitor_base();
  BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
  BasicObjectLock* entry = NULL;
  while (most_recent != limit ) {
    if (most_recent->obj() == NULL) entry = most_recent;
    else if (most_recent->obj() == lockee) break;
    most_recent++;
  }
  
  // 找到 Lock Record
  if (entry != NULL) {
    // 对 Lock Record 的 obj 字段赋值
    entry->set_obj(lockee);
    int success = false;
    uintptr_t epoch_mask_in_place = (uintptr_t)markOopDesc::epoch_mask_in_place;
    
    markOop mark = lockee->mark();
    intptr_t hash = (intptr_t) markOopDesc::no_hash;
    
    // ## 2. 根据 mark word 的最低3位是否“101”，判断是否使用偏向锁
    if (mark->has_bias_pattern()) {
      uintptr_t thread_ident;
      uintptr_t anticipated_bias_locking_value;
      thread_ident = (uintptr_t)istate->thread();
      
      // ## 3. 对偏向锁相关的bit进行计算，分为如下4步：
      //  （1）将 prototype_header 与线程ID “相或” 操作，得到：【线程ID + prototype_header 中的（epoch + 分代年龄 + 偏向锁标志 +锁标志位）】，分代年龄为 0。
      //  （2）将（1）的结果与锁对象的 markOop 进行“异或”，相等的位全部被置为0，只剩下不相等的位。
      //  （3）对 age_mask_in_place “取反”，得到一个 MarkWord 结构的数组，除了分代年龄的4位为0，其他位为1。
      //  （4）将（2）和（3）的结果进行 “相与”操作，把（2）的结果关于分代年龄的bit忽略掉。
      anticipated_bias_locking_value =
             (((uintptr_t)lockee->klass()->prototype_header() |thread_ident) ^ (uintptr_t)mark) &
             ~((uintptr_t) markOopDesc::age_mask_in_place);
    
      // ## 4. anticipated_bias_locking_value = 0，表示：
      //  （1）锁对象的 Mark Word的 线程ID 指向当前线程
      //  （2）锁对象的 Mark Word的 Epoch == klass 的 Epoch
      // 结论是 【当前线程获得偏向锁】，什么都不用做，返回
      if  (anticipated_bias_locking_value == 0) {
        // already biased towards this thread, nothing to do
        if (PrintBiasedLockingStatistics) {
          (* BiasedLocking::biased_lock_entry_count_addr())++;
        }
        success = true;
      }
      // 来到这里，表示【锁对象并没有偏向当前线程】，判断是否需要【撤销锁对象的偏向】。
     
      // ## 5. 将 anticipated_bias_locking_value 与 biased_lock_mask_in_place（111）相与。
      //  （1）如果结果 != 0，说明 locking_value 的后3位存在不为0的位
      //  （2）说明之前 “异或” 操作时，类的 prototype_header 与锁对象 markOop 的后3位不相等
      //  （3）代码走到这里，markOop 的后3位一定是 “101”（即偏向模式）
      //  （4）就是类的 prototype_header 的后3位不是 “101”，即【对象所属类不再支持偏向】，发生了 【bulk_revoke】。
      // 结论是 【需要对当前对象进行偏向锁的撤销】
      else if ((anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0) {
        // try revoke bias
        markOop header = lockee->klass()->prototype_header();
        if (hash != markOopDesc::no_hash) {
          header = header->copy_set_hash(hash);
        }
        // 利用 CAS 操作将 mark word 替换为 klass 中的 mark word
        if (Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark) {
          if (PrintBiasedLockingStatistics)
            (*BiasedLocking::revoked_lock_entry_count_addr())++;
        }
      }
      
      // 来到这里，表示【类还支持偏向锁】，需要【判断当前对象的epoch】是否合法，如果不合法，需要进行【重偏向】
      
      ## 6. 将 locking_value 与 epoch_mask_in_place 相与，检查类的 prototype_header 中 epoch 是否为0。
      //  （1）如果结果 != 0，说明类的 prototype_header 中 epoch 和对象 markOop 的 epoch 不相等。
      //  （2）说明类在对象分配后发生过 【bulk_rebais】，所以之前对象的偏向就无效了
      //  （3）每次发生【bulk_rebaise】,类的 prototype header 中的 epoch 都会+1
      // 结论是：需要进行【重偏向】
      else if ((anticipated_bias_locking_value & epoch_mask_in_place) !=0) {
        // try rebias
        // 构造一个偏向当前线程的mark word
        markOop new_header = (markOop) ( (intptr_t) lockee->klass()->prototype_header() | thread_ident);
        if (hash != markOopDesc::no_hash) {
          new_header = new_header->copy_set_hash(hash);
        }
        // CAS替换对象头的mark word
        if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), mark) == mark) {
          if (PrintBiasedLockingStatistics)
            (* BiasedLocking::rebiased_lock_entry_count_addr())++;
        }
        // 重偏向失败，代表存在多线程竞争，则调用monitorenter方法进行锁升级
        else {
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
        success = true;
      }
      
      // 来到这里，表示走到这里说明当前要么偏向别的线程，要么是匿名偏向（即没有偏向任何线程）。可以尝试获取锁。
      
      else {
      
        // try to bias towards thread in case object is anonymously biased
        // 构建一个匿名偏向的mark word，尝试用 CAS 指令替换掉锁对象的 mark word
        markOop header = (markOop) ((uintptr_t) mark & ((uintptr_t)markOopDesc::biased_lock_mask_in_place |
                                                              (uintptr_t)markOopDesc::age_mask_in_place |
                                                              epoch_mask_in_place));
        if (hash != markOopDesc::no_hash) {
          header = header->copy_set_hash(hash);
        }
        markOop new_header = (markOop) ((uintptr_t) headerthread_ident);
        if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), header) == header) {
          if (PrintBiasedLockingStatistics)
            BiasedLocking::anonymously_biased_lock_entry_count_add++;
        }
        else {
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entryhandle_exception);
        }
        success = true;
      }
    }
    
    
    // traditional lightweight locking
    // 这里开始是轻量级锁的逻辑了，省略
    if (!success) {
        // ... 
    }
  }
}
```


### 偏向锁撤销
<font color=orange>撤销是指在获取偏向锁的过程因为不满足条件导致要将锁对象改为非偏向锁状态</font>。

前面提到过，当获取偏向锁失败时，会进入到`InterpreterRuntime::monitorenter`方法：
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
如果开启偏向锁，进入`fast_enter`流程：
```
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
  if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond =BiasedLocking::revoke_and_rebias(obj,  attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by  now");
  }

  slow_enter (obj, lock, THREAD) ;
}
```
* 如果不在 safepoint ，那么应该是Java线程，尝试 <font color=green>**撤销并重偏向锁**</font>。
  * 如果成功，直接返回。
  * 否则，进入到`slow_enter`流程。
* 如果在 safepoint ，那么应该是VM线程，尝试 <font color=green>**撤销偏向锁**</font>。


看到`revoke_and_rebias`方法：
```
BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {

  assert(!SafepointSynchronize::is_at_safepoint(), "must not be called while at safepoint");
  
  markOop mark = obj->mark();
  
  // ## 1. 如果当前锁是【匿名偏向】 并且 【不是要获取偏向锁】
  // -------->  如锁对象的hashcode方法被调用会出现这种情况，需要撤销偏向锁。 <-----
  if (mark->is_biased_anonymously() && !attempt_rebias) {
    markOop biased_value       = mark;
    markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
    markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
    if (res_mark == biased_value) {
      return BIAS_REVOKED;
    }
  }
  
  // ## 2. 如果锁对象开启偏向锁
  else if (mark->has_bias_pattern()) {
    Klass* k = obj->klass();
    markOop prototype_header = k->prototype_header();
    
    // ## 2.1 如果对应的 klass 关闭了偏向锁，即发生了 【bulk_revoke】
    // 这时，只需要通过 CAS 设置对象的 Mark Word（为 klass 的 mark word）。不需要记录锁撤销。
    // 即使 CAS 操作失败，说明【有其他线程将它撤销了】，于是直接返回。
    if (!prototype_header->has_bias_pattern()) {
      markOop biased_value       = mark;
      markOop res_mark = (markOop) Atomic::cmpxchg_ptr(prototype_header, obj->mark_addr(), mark);
      assert(!(*(obj->mark_addr()))->has_bias_pattern(), "even if we raced, should still be revoked");
      return BIAS_REVOKED;
    }
    
    // ## 2.2 如果锁对象的epoch过期了，意味着这个对象的状态是【未偏向】的
    // 因此可以直接通过 CAS 操作设置对象的 Mark Word。不需要记录锁撤销。
    else if (prototype_header->bias_epoch() != mark->bias_epoch()) {
    
      // ## 2.2.1 将锁重偏向为当前线程
      if (attempt_rebias) {
        assert(THREAD->is_Java_thread(), "");
        markOop biased_value       = mark;
        markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(rebiased_prototype, obj->mark_addr(), mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED_AND_REBIASED;
        }
      } 
      // ## 2.2.2 将锁撤销
      else {
        markOop biased_value       = mark;
        markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED;
        }
      }
    }
  }
  
  
  // ## 3. 更新偏向锁撤销的次数（与批量重偏向与批量撤销相关）
  HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
  // ## 3.1 当前锁对象状态为【不可偏向】
  if (heuristics == HR_NOT_BIASED) {
    return NOT_BIASED;
  }
  
  // ## 3.2 撤销单个线程获得的偏向锁
  else if (heuristics == HR_SINGLE_REVOKE) {
    Klass *k = obj->klass();
    markOop prototype_header = k->prototype_header();
    
    // ## 3.2.1 如果撤销的是当前线程自己的偏向锁（比如调用 hashcode()方法导致）
    // 只需要直接修改锁对象的 Mark Word 即可。 
    if (mark->biased_locker() == THREAD &&
        prototype_header->bias_epoch() == mark->bias_epoch()) {
      BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD);
      ((JavaThread*) THREAD)->set_cached_monitor_info(NULL);
      assert(cond == BIAS_REVOKED, "why not?");
      return cond;
    }
    
    // ## 3.2.2 如果撤销的是其他线程拥有的偏向锁，就需要通过VM线程撤销偏向锁
    else {
      VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
      VMThread::execute(&revoke);
      return revoke.status_code();
    }
  }
  
  
  assert((heuristics == HR_BULK_REVOKE) ||
         (heuristics == HR_BULK_REBIAS), "?");
         
  // ## 4. 通过VM线程执行【批量撤销、批量重偏向】的逻辑
  VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                (heuristics == HR_BULK_REBIAS),
                                attempt_rebias);
  VMThread::execute(&bulk_revoke);
  return bulk_revoke.status_code();
}
```
这个方法的逻辑可以总结为：
* 之所以会进入到这个`revoke_and_rebias`方法，是由于获取偏向锁失败了（出现线程竞争），于是需要撤销偏向锁。（由于支持==批量撤销、批量重偏向==的关系，使得这个方法变复杂了）
* 先处理以下情况，它们都是不需要记录锁撤销的：
  * 如果 klass 关闭了偏向锁，即发生了【批量撤销】：设置对象mark word为klass的值，然后返回。
  * 锁对象的epoch过期，即锁对象是未偏向的：只要通过CAS设置对象的 Mark Word（将锁重偏向或将锁撤销），然后返回
* 记录偏向锁撤销的次数，根据撤销次数与相关阈值，作如下处理：
  * 撤销单个线程获得的偏向锁
    * 如果是当前线程自己的偏向锁，直接执行偏向锁撤销逻辑
    * 如果是其他线程的偏向锁，需要由VM线程执行
  * 批量撤销、批量重偏向：由VM线程执行
  

撤销偏向锁的逻辑在 **revoke_bias** 这个方法实现：
```
static BiasedLocking::Condition revoke_bias(oop obj, bool allow_rebias, bool is_bulk, JavaThread* requesting_thread) {

  markOop mark = obj->mark();
  if (!mark->has_bias_pattern()) {
    return BiasedLocking::NOT_BIASED;
  }
  
  
  uint age = mark->age();
  // 构建两个mark word，一个是匿名偏向模式（101），一个是无锁模式（001）
  markOop   biased_prototype = markOopDesc::biased_locking_prototype()->set_age(age);
  markOop unbiased_prototype = markOopDesc::prototype()->set_age(age);

  JavaThread* biased_thread = mark->biased_locker();
  
  // ## 1. 匿名偏向（例如调用 hashCode 方法会来到这里）
  if (biased_thread == NULL) {
  
    // ## 1.1 如果不允许重偏向，则将对象的mark word设置为无锁模式
    if (!allow_rebias) {
      obj->set_mark(unbiased_prototype);
    }
    
    return BiasedLocking::BIAS_REVOKED;
  }
  
  
  // ## 2. 判断拥有偏向锁的线程是否存活
  bool thread_is_alive = false;
  // ## 2.1 如果是当前线程，自然是存活的
  if (requesting_thread == biased_thread) {
    thread_is_alive = true;
  }
  // ## 2.2 遍历JVM的所有线程去找
  else {
    for (JavaThread* cur_thread = Threads::first(); cur_thread != NULL; cur_thread = cur_thread->next()) {
      if (cur_thread == biased_thread) {
        thread_is_alive = true;
        break;
      }
    }
  }
  
  // ## 3. 如果线程已经不存活了
  if (!thread_is_alive) {
    // 允许重偏向则将对象mark word设置为匿名偏向状态，否则设置为无锁状态
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    }
    else {
      obj->set_mark(unbiased_prototype);
    }
    return BiasedLocking::BIAS_REVOKED;
  }
  
  
  // ## 4. 线程还存活，需要遍历线程栈中所有的【Lock Record】
  
  // ## 4.1 从线程的栈帧中取出 monitors 数组
  GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(biased_thread);
  BasicLock* highest_lock = NULL;
  for (int i = 0; i < cached_monitor_info->length(); i++) {
    MonitorInfo* mon_info = cached_monitor_info->at(i);
    
    // ## 4.2 如果找到对应的 Lock Record，说明【这个线程还在执行同步块中的代码】
    if (mon_info->owner() == obj) {
      
      // ## 4.3 【需要升级为轻量级锁】，直接修改此 Lock Record
      // 处理锁重入的case，需注意2点：
      //   （A）这里的 for 循环是对每一个 Lock Record 都这样操作的（没有 break）。
      //   （B）将 Lock Record 的【 Displaced Mark Word】设置为null
      //   （B）第一个Lock Record会在下面的代码中再处理
      markOop mark = markOopDesc::encode((BasicLock*) NULL);
      highest_lock = mon_info->lock();
      highest_lock->set_displaced_header(mark);
    }
  }
  
  if (highest_lock != NULL) {
    // ## 4.4 修改第一个Lock Record为无锁状态，然后将obj的mark word设置为指向该Lock Record的指针
    highest_lock->set_displaced_header(unbiased_prototype);
    obj->release_set_mark(markOopDesc::encode(highest_lock));
  }
  
  // ## 5. 偏向线程已经不在同步块中了，将偏向锁设置为【匿名偏向】或【无锁】状态
  else {
    if (allow_rebias) {
      obj->set_mark(biased_prototype);
    } else {
      // Store the unlocked value into the object's header.
      obj->set_mark(unbiased_prototype);
    }
  }
  
  return BiasedLocking::BIAS_REVOKED;
}
```
**小结**：revoke_bias--撤销偏向锁
撤销偏向锁的逻辑：
1. 查看偏向的线程是否存活，如果不存活了，则直接撤销偏向锁。
2. 偏向的线程是否还在同步块中，如果不在了，则撤销偏向锁。
3. 将偏向线程所有相关 Lock Record 的 Displaced Mark Word 设置为null（因为锁重入，会导致Lock Record有多个）。
4. 将第一个 Lock Record 的 Displaced Mark Word  设置为无锁状态，然后将对象头指向这个 Lock Record（这里不需要用CAS指令，因为是在safepoint）。
5. 此时，已经升级成了轻量级锁。



### 偏向锁释放
入口在 bytecodeInterpreter.cpp：
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
偏向锁释放就直接把 Lock Record 释放掉就可以了。


### 批量重偏向和批量撤销
在前面提到过：
* 每次撤销偏向锁的时候都通过`update_heuristics`方法记录下来
* 以类为单位，当某个类的对象撤销偏向次数达到一定阈值的时候JVM就认为该类不适合偏向模式或者需要重新偏向另一个对象
* 通过 VM Thread 在 【safepoint】时进行【批量撤销】或【批量重偏向】

VM Thread调用下面这个方法：
```
static BiasedLocking::Condition bulk_revoke_or_rebias_at_safepoint(oop o,
                                                                   bool bulk_rebias,
                                                                   bool attempt_rebias_of_object,
                                                                   JavaThread* requesting_thread) {
  
  // ... 省略
  
  jlong cur_time = os::javaTimeMillis();
  o->blueprint()->set_last_biased_lock_bulk_revocation_time(cur_time);
  
  klassOop k_o = o->klass();
  Klass* klass = Klass::cast(k_o);
  
  // ## 1. 批量重偏向
  if (bulk_rebias) {
  
    // ## 1.1 只有 klass 的 markOop 是偏向模式，才去更新 【epoch】 字段
    //  如果不是偏向模式，说明有其他的 【VM operation】导致这个 markOop 被修改。
    if (klass->prototype_header()->has_bias_pattern()) {
    
      // ## 1.2 【epoch】自增
      int prev_epoch = klass->prototype_header()->bias_epoch();
      klass->set_prototype_header(klass->prototype_header()->incr_bias_epoch());
      int cur_epoch = klass->prototype_header()->bias_epoch();
      
      
      // ## 1.3 遍历所有线程
      for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
      
        // ## 1.3.1 获取此线程的所有 Lock Record
        GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
        
        // ## 1.3.2 遍历此线程的所有 Lock Record
        for (int i = 0; i < cached_monitor_info->length(); i++) {
          MonitorInfo* mon_info = cached_monitor_info->at(i);
          oop owner = mon_info->owner();
          markOop mark = owner->mark();
          
          // ## 1.3.3 如果 Lock Record 关联的锁对象是这个 klass 类型的，更新锁对象的【epoch】
          if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
            // We might have encountered this object already in the case of recursive locking
            assert(mark->bias_epoch() == prev_epoch || mark->bias_epoch() == cur_epoch, "error in bias epoch adjustment");
            owner->set_mark(mark->set_bias_epoch(cur_epoch));
          }
        }
      }
    }
    
    // 撤销这个锁对象的偏向锁
    revoke_bias(o, attempt_rebias_of_object && klass->prototype_header()->has_bias_pattern(), true, requesting_thread);
  }
  
  // ## 2. 批量撤销
  else {
  
    // ## 2.1 关闭这个 klass 的偏向锁，会导致后面：
    //   （1） 分配属于这个 klass 的实例，处于【无锁模式】
    //   （2） 获取这个对象的锁时，获取轻量级锁
    klass->set_prototype_header(markOopDesc::prototype());
    
    // ## 2.2 遍历所有线程的栈，对于已存在的这个 klass 的已偏向实例，撤销它们的偏向锁
    for (JavaThread* thr = Threads::first(); thr != NULL; thr = thr->next()) {
      GrowableArray<MonitorInfo*>* cached_monitor_info = get_or_compute_monitor_info(thr);
      for (int i = 0; i < cached_monitor_info->length(); i++) {
        MonitorInfo* mon_info = cached_monitor_info->at(i);
        oop owner = mon_info->owner();
        markOop mark = owner->mark();
        if ((owner->klass() == k_o) && mark->has_bias_pattern()) {
          revoke_bias(owner, false, true, requesting_thread);
        }
      }
    }
    
     // 撤销这个锁对象的偏向锁
    revoke_bias(o, false, true, requesting_thread);
  }
}                                                                   
```


### 总结

#### 偏向锁的目的
对于场景为：锁不但没有多线程竞争，而且总是由同一个线程获取，使用偏向锁可以提高性能。

#### 偏向锁的获取
Java线程执行字节码`monitorenter`，开始同步块的访问，流程如下：
* (1) 先检查对象头 Mark Word 中的【bias_pattern】是否为 “101”
  * 如果是（即处于【偏向锁状态】），进入【偏向锁的获取】的流程
  * 如果不是，进入【轻量级锁获取的流程】
* (2) 检查对象头 Mark Word 中记录的 Thread ID 是否是当前线程ID
  * 如果是：说明当前线程已经获得此对象的偏向锁（锁重入）
    * 只需要在栈添加一条 Lock Record（类型为`MonitorInfo`），不需要 CAS 操作，因为操作的是自己的栈
    * 不需要操作锁对象
  * 如果不是，进行一些检查（主要是为了支持“批量重偏向和批量撤销”）
* (3) 检查对象所属类是否不再支持偏向
  * 是由于发生了【批量撤销】导致的
  * 进入到【获取轻量级锁】的流程
* (4) 检查当前对象的 Epoch 是否为不合法（不等于 klass 的 Epoch）
  * 是由于发生了【批量重偏向】导致的
  * 说明对象之前的偏向是无效了，需要【重偏向】
  * 尝试使用 CAS 替换对象头 Mark Word 中的 Thread ID 为当前线程ID
    * 如果成功， 表示获取到偏向锁
    * 如果失败，需要进入【获取轻量级锁】的流程
* (5) 通过CAS 操作，将当前线程ID替换进对象的 Mark Word，此时有2种情况：
    * 当前对象处于【匿名偏向锁状态（可偏向未锁定）】，这种情况会替换成功（即获取到锁），而且同样会在栈添加一条 Lock Record。
    * 偏向锁被其他线程拥有，这种情况会替换失败，开始进行【偏向锁撤销】。这个是偏向锁的特点：等到出现竞争才释放锁。

#### 偏向锁的撤销
<font color=orange>撤销是指在获取偏向锁的过程因为不满足条件导致要将锁对象改为非偏向锁状态</font>。

流程如下：
* (1) 如果 klass 关闭了偏向锁，即发生了 【批量撤销】，使用 CAS 设置对象的 Mark Word，然后返回。
* (2) 如果锁对象的epoch过期了，意味着这个对象的状态是【未偏向】的：
  * 直接通过 CAS 操作设置对象的 Mark Word 将锁撤销或重偏向
  * 不需要记录锁撤销
* (3) 更新这个 klass 的偏向锁撤销的次数，和相关阈值比较，有2种情况：
  * 撤销单个线程获得的偏向锁
    * 如果是当前线程自己的偏向锁，直接执行偏向锁撤销逻辑
    * 如果是其他线程的偏向锁，需要由VM线程执行
  * 批量撤销、批量重偏向：由VM线程执行
* (4) 偏向锁撤销逻辑：
  * 查看偏向的线程是否存活，如果不存活了，则直接撤销偏向锁。
  * 偏向的线程是否还在同步块中，如果不在了，则撤销偏向锁。
  * 将偏向线程所有相关 Lock Record 的 Displaced Mark Word 设置为null（因为锁重入，会导致Lock Record有多个）。
  * 将第一个 Lock Record 的 Displaced Mark Word  设置为无锁状态，然后将对象头指向这个 Lock Record（这里不需要用CAS指令，因为是在safepoint）。




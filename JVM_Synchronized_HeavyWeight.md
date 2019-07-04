---
layout: post
title: "Synchronized底层实现—-重量级锁（未完）"
date: 2019-06-26 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

学习JVM源码Synchronized锁部分，写了如下几篇文章：
* Synchronized底层实现—-概论
* Synchronized底层实现—-偏向锁
* Synchronized底层实现—-轻量级锁
* Synchronized底层实现—-重量级锁（本文）

<!--more-->


### 锁膨胀

#### <font color=red>锁膨胀发生的时机？</font>
目前已知的是轻量级锁膨胀为重量级锁，还有其他场景？<font color=red>待解决</font>！


#### 锁膨胀的逻辑
```
ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {

  for (;;) {
    const markOop mark = object->mark() ;
    assert (!mark->has_bias_pattern(), "invariant") 
    
    // The mark can be in one of the following states:
    // *  Inflated（重量级锁状态）      - just return（直接返回）
    // *  Stack-locked（轻量级锁状态）  - coerce it to inflated（膨胀）
    // *  INFLATING（膨胀中）          - busy wait for conversion tcomplete（忙等待直到膨胀完成）
    // *  Neutral（无锁状态）          - aggressively inflate thobject（膨胀）
    // *  BIASED（偏向锁）             - Illegal.  We should never sethis（非法状态，在这里不会出现
    
    
    // ## 1. 状态是【inflated】，已经是重量级锁状态了，直接返回
    if (mark->has_monitor()) {
        ObjectMonitor * inf = mark->monitor() ;
        assert (inf->header()->is_neutral(), "invariant");
        assert (inf->object() == object, "invariant") ;
        assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitois invalid");
        return inf ;
    }
    
    // ## 2. 状态是【inflation in progress】，锁膨胀中。
    // （1）说明当前有其他线程在膨胀这个锁
    // （2）只有一个线程可以膨胀锁，其他线程必须等待
    if (mark == markOopDesc::INFLATING()) {
       TEVENT (Inflate: spin while INFLATING) ;
       ReadStableMark(object) ;
       continue ;
    }
    
    // ## 3. 状态是【stack-locked】，轻量级锁状态，需要锁膨胀。
    if (mark->has_locker()) {
      // ## 3.1 分配并初始化一个 ObjectMonitor 对象
      ObjectMonitor * m = omAlloc (Self) ;
      m->Recycle();
      m->_Responsible  = NULL ;
      m->OwnerIsThread = 0 ;
      m->_recursions   = 0 ;
      m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;   /Considermaintain by type/class
    
      // ## 3.2 使用 CAS 将锁对象的 Mark Word 设置为 【INFLATING】状态
      // 用于表示当前这个 Mark Word 是 【Busy】
      markOop cmp = (markOop) Atomic::cmpxchg_pt(markOopDesc::INFLATING(), object->mark_addr(), mark) ;
      if (cmp != mark) {
        omRelease (Self, m, true) ;
        continue ;       // Interference -- just retry
      }
      
      // ## 3.3 取出保存在栈中 Lock Record 的【displaced mark word】
      markOop dmw = mark->displaced_mark_helper() ;
      assert (dmw->is_neutral(), "invariant") ;
      
      // ## 3.4 设置 monitor 的字段
      m->set_header(dmw) ;
      m->set_owner(mark->locker());
      m->set_object(object);
      
      guarantee (object->mark() == markOopDesc::INFLATING(), "invariant";
      
      // ## 3.5 将锁对象头设置为【重量级锁】状态
      object->release_set_mark(markOopDesc::encode(m));
      
      // ... 省略
      
      return m ;
    }
    
    
    // ## 4. 状态是【neutral】，无锁状态，需要膨胀
    assert (mark->is_neutral(), "invariant");
    
    // ## 4.1 分配并初始化一个 ObjectMonitor 对象
    ObjectMonitor * m = omAlloc (Self) ;
    m->Recycle();
    m->set_header(mark);
    m->set_owner(NULL);
    m->set_object(object);
    m->OwnerIsThread = 1 ;
    m->_recursions   = 0 ;
    m->_Responsible  = NULL ;
    m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;       // considerkeep metastats by type/class
    
    // ## 4.2 使用 CAS 将锁对象的 Mark Word 设置为【重量级锁状态】
    if (Atomic::cmpxchg_ptr (markOopDesc::encode(m), object->mark_addr()mark) != mark) {
      m->set_object (NULL) ;
      m->set_owner  (NULL) ;
      m->OwnerIsThread = 0 ;
      m->Recycle() ;
      omRelease (Self, m, true) ;
      m = NULL ;
      continue ;
      // interference - the markword changed - just retry.
      // The state-transitions are one-way, so there's no chance of
      // live-lock -- "Inflated" is an absorbing state.
    }
    
    // ... 省略
    
    return m ;
}
```
总结这个方法：
* 锁在这个方法完成“膨胀”
* 如果有多个线程同时执行“膨胀”操作，只有一个线程可以执行，其他线程必须等待
* 需要膨胀的锁会有如下不同状态：
  * Inflated（重量级锁状态）：直接返回
  * Stack-locked（轻量级锁状态）： 进行锁膨胀
  * INFLATING（膨胀中）：忙等待直到完成（其他线程执行完膨胀逻辑）
  * Neutral（无锁状态）：进行锁膨胀
* 对于轻量锁的膨胀，分为如下几步：
  * 分配并初始化一个 ObjectMonitor 对象
  * 将锁对象设置为【INFLATING】状态
  * 设置 ObjectMonitor 对象：
    * `header`字段设置为保存在栈中 Lock Record 的【displaced mark word】（即原来锁对象的 mark word）
    * `owner`字段设置为栈中 Lock Record 的地址
    * `object`字段设置为锁对象
  * 设置锁对象的 mark word 为重量级锁



### 获取锁
获取锁的逻辑如下：
```
void ATTR ObjectMonitor::enter(TRAPS) {

  Thread * const Self = THREAD ;
  void * cur ;
  
  // ## 1. 使用 CAS 设置当前线程到 _owner 字段，如果成功，那么当前线程获得锁
  // 如果 cur（_owner 原来的值） 为 NULL，表示【原来是无锁状态】
  cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
  if (cur == NULL) {
    assert (_recursions == 0   , "invariant") ;
    assert (_owner      == Self, "invariant") ;
    return ;
  }
  
  // ## 2.【锁重入】的情况，直接返回
  if (cur == Self) {
    _recursions ++ ;
    return ;
  }
  
  // ## 3. 如果 cur 是当前线程的一个锁记录的地址
  //  （1）在轻量级锁【膨胀】时，会将 _owner 字段设置为锁记录的地址
  //  （2）因此，这里可以判断【当前线程是否原来持有轻量级锁的线程】
  //  （3）设置 _owner 字段为当前线程
  if (Self->is_lock_owned ((address)cur)) {
    _recursions = 1 ;
    _owner = Self ;
    OwnerIsThread = 1 ;
    return ;
  }
  
  // -------->  来到这里说明遇到锁竞争了  <------------
  
  // ## 4. 先尝试自旋获取锁
  // （1）主要是用 CAS 将当前线程设置到 _owner 字段。
  // （2）会检查是否有GC进行，如果有，跳出自旋。
  // （3）如果自旋得到锁，返回
  if (Knob_SpinEarly && TrySpin (Self) > 0) {
    Self->_Stalled = 0 ;
    return ;
  }
  
  // ... 省略
  
  {
    // ## 5. 设置 Java Thread 和关联的 osThread 的状态
    JavaThreadBlockedOnMonitorEnterState jtbmes(jt, this);
    OSThreadContendState osts(Self->osthread());
    ThreadBlockInVM tbivm(jt);
    
      
    // ## 6. 
    for (;;) {
      // 这里和 【park、unpark】相关
      jt->set_suspend_equivalent();
      
      EnterI (THREAD) ;
      
      if (!ExitSuspendEquivalent(jt)) break ;
      
      _recursions = 0 ;
    
    }
      
      
  }
  
  
}
```
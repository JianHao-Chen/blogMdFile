---
layout: post
title: "HotSpot老年代TenuredGeneration的实现"
date: 2019-06-19 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文研究HotSpot老年代TenuredGeneration的实现。

<!--more-->

## TenuredGeneration的GC实现

老年代TenuredGeneration所使用的垃圾回收算法是**标记-压缩-清理**算法。

GC的入口：
```
void TenuredGeneration::collect(bool   full,
                                bool   clear_all_soft_refs,
                                size_t size,
                                bool   is_tlab) {
  retire_alloc_buffers_before_full_gc();
  OneContigSpaceCardGeneration::collect(full, clear_all_soft_refs,
                                        size, is_tlab);
}
```
实际收集工作在父类的collect方法：
```
void OneContigSpaceCardGeneration::collect(bool   full,
                                           bool   clear_all_soft_refs,
                                           size_t size,
                                           bool   is_tlab) {
  SpecializationStats::clear();
  // Temporarily expand the span of our ref processor, so
  // refs discovery is over the entire heap, not just this generation
  ReferenceProcessorSpanMutator
    x(ref_processor(), GenCollectedHeap::heap()->reserved_region());
  GenMarkSweep::invoke_at_safepoint(_level, ref_processor(), clear_all_soft_refs);
  SpecializationStats::print();
}
```
重点关注到`GenMarkSweep::invoke_at_safepoint`方法的实现。

### 准备工作
```
  // 设置引用处理器和引用的处理策略；
  _ref_processor = rp;
  rp->setup_policy(clear_all_softrefs);

  // 设置输出日志；
  TraceTime t1("Full GC", PrintGC && !PrintGCDetails, true, gclog_or_tty);

  // When collecting the permanent generation methodOops may be moving,
  // so we either have to flush all bcp data or convert it into bci.
  CodeCache::gc_prologue();
  Threads::gc_prologue();

  // 增加永久代回收的统计次数
  // Increment the invocation count for the permanent generation, since it is
  // implicitly collected whenever we do a full mark sweep collection.
  gch->perm_gen()->stat_record()->invocations++;
  
  // 统计GC前的内存堆已使用大小
  // Capture heap size before collection for printing.
  size_t gch_prev_used = gch->used();

  // 保存当前内存代和更低的内存代、以及永久代的已使用区域
  // Capture used regions for each generation that will be
  // subject to collection, so that card table adjustments can
  // be made intelligently (see clear / invalidate further below).
  gch->save_used_regions(level, true /* perm */);
  
  // 初始化遍历栈，用来保存对象和对象头的对应关系
  allocate_stacks();
```

### 执行GC
整个过程一共4阶段，分别对应4个方法的实现：
```
  // Mark live objects
  static void mark_sweep_phase1(int level, bool clear_all_softrefs);
  // Calculate new addresses
  static void mark_sweep_phase2();
  // Update pointers
  static void mark_sweep_phase3(int level);
  // Move objects to new positions
  static void mark_sweep_phase4();
```

#### mark_sweep_phase1 -- 递归标记GC Root存活对象

##### 标记根集对象
```
GenCollectedHeap* gch = GenCollectedHeap::heap();
follow_root_closure.set_orig_generation(gch->get_gen(level));
gch->gen_process_strong_roots(level,
                                false, // Younger gens are not roots.
                                true,  // activate StrongRootsScope
                                true,  // Collecting permanent generation.
                                SharedHeap::SO_SystemClasses,
                                &follow_root_closure,
                                true,   // walk code active on stacks
                                &follow_root_closure);
```
* `GenCollectedHeap::gen_process_strong_roots`这个方法在讲新生代回收时讲过，就是遍历根集对象。
* 对每一个遍历的对象，通过`MarkSweep::FollowRootClosure`处理。

看到`FollowRootClosure::follow_root`的实现：
```
template <class T> inline void MarkSweep::follow_root(T* p) {
  // ...省略
  
  T heap_oop = oopDesc::load_heap_oop(p);
  if (!oopDesc::is_null(heap_oop)) {
    oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);
    if (!obj->mark()->is_marked()) {
      mark_object(obj);
      obj->follow_contents();
    }
  }
  follow_stack();
}
```
* 如果对象没有被标记
  * 调用`mark_object`标记对象。如果需要，保存对象头（根据是否使用偏向锁、是否上锁等）。
  * 调用`follow_contents`方法（具体看到`instanceKlass::oop_follow_contents`方法），遍历这个oop和它所引用的所有对象，将它们标记并压栈`_marking_stack`。
* 调用`follow_stack`方法，出栈`_marking_stack`里面的对象，调用`follow_contents`方法。

到此，所有的根集对象都被标记了。


##### 处理在标记过程中发现的引用
```
  ref_processor()->setup_policy(clear_all_softrefs);
  ref_processor()->process_discovered_references(
      &is_alive, &keep_alive, &follow_stack_closure, NULL);
```

##### 卸载不再使用的类，并清理CodeCache和标记栈
```
// Follow system dictionary roots and unload classes
bool purged_class = SystemDictionary::do_unloading(&is_alive);

// Follow code cache roots
CodeCache::do_unloading(&is_alive, &keep_alive, purged_class);
follow_stack(); // Flush marking stack
```

##### 更新存活类的子类、兄弟类、实现类的引用关系，清理未被标记的软引用和弱引用
```
follow_weak_klass_links();

follow_mdo_weak_refs();
```

##### 清理字符串常量池中没有被标记过的对象
```
StringTable::unlink(&is_alive);
```

##### 清理符号表中没有被引用的符号
```
SymbolTable::unlink();
```

#### mark_sweep_phase2 -- 计算活跃对象在压缩完成之后的新地址

```
void GenMarkSweep::mark_sweep_phase2() {
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  Generation* pg = gch->perm_gen();
  // ... 省略
  
  gch->prepare_for_compaction();

  CompactPoint perm_cp(pg, NULL, NULL);
  pg->prepare_for_compaction(&perm_cp);
}
```

先看到`GenCollectedHeap::prepare_for_compaction`方法，这是对整个堆做压缩的准备工作：
```
void GenCollectedHeap::prepare_for_compaction() {
  Generation* scanning_gen = _gens[_n_gens-1];
  // Start by compacting into same gen.
  CompactPoint cp(scanning_gen, NULL, NULL);
  while (scanning_gen != NULL) {
    scanning_gen->prepare_for_compaction(&cp);
    scanning_gen = prev_gen(scanning_gen);
  }
}
```
可以看到，分别调用它各个分代的`prepare_for_compaction`方法：
```
void Generation::prepare_for_compaction(CompactPoint* cp) {
  // Generic implementation, can be specialized
  CompactibleSpace* space = first_compaction_space();
  while (space != NULL) {
    space->prepare_for_compaction(cp);
    space = space->next_compaction_space();
  }
}
```
相似的，调用各个Space的`prepare_for_compaction`方法。由于TenuredGeneration只有一个`TenuredSpace`，找到这个方法的实现：
```
void ContiguousSpace::prepare_for_compaction(CompactPoint* cp) {
  SCAN_AND_FORWARD(cp, top, block_is_always_obj, obj_size);
}
```
这个宏`SCAN_AND_FORWARD`做的是：==为活跃对象计算新地址并保存在对象头==。



#### mark_sweep_phase3 -- 更新对象的引用地址
```
  adjust_root_pointer_closure.set_orig_generation(gch->get_gen(level));
  adjust_pointer_closure.set_orig_generation(gch->get_gen(level));

  gch->gen_process_strong_roots(level,
                                false, // Younger gens are not roots.
                                true,  // activate StrongRootsScope
                                true,  // Collecting permanent generation.
                                SharedHeap::SO_AllClasses,
                                &adjust_root_pointer_closure,
                                false, // do not walk code
                                &adjust_root_pointer_closure);
```
adjust_root_pointer_closure和adjust_pointer_closure都是静态创建的对象引用地址调整函数的封装对象，这里将调用gen_process_strong_roots()并使用这两个处理函数调整根集对象指针的引用地址。

`AdjustPointerClosure`的工作函数：
```
void MarkSweep::AdjustPointerClosure::do_oop(oop* p)       { adjust_pointer(p, _is_root); }
void MarkSweep::AdjustPointerClosure::do_oop(narrowOop* p) { adjust_pointer(p, _is_root); }
```
继续看`MarkSweep::adjust_pointer`方法：
```
template <class T> inline void MarkSweep::adjust_pointer(T* p, bool isroot) {
  T heap_oop = oopDesc::load_heap_oop(p);
  if (!oopDesc::is_null(heap_oop)) {
    oop obj     = oopDesc::decode_heap_oop_not_null(heap_oop);
    oop new_obj = oop(obj->mark()->decode_pointer());
    
    // ... assert
    
    if (new_obj != NULL) {
      assert(Universe::heap()->is_in_reserved(new_obj),
             "should be in object space");
      oopDesc::encode_store_heap_oop_not_null(p, new_obj);
    }
  }
}
```
这个方法的逻辑是：解析引用对象的MarkWord，若该引用对象已经被标记，就会解析转发指针，并设置引用地址为引用对象新的地址。



#### mark_sweep_phase4 -- 移动所有活跃对象到新地址
```
void GenMarkSweep::mark_sweep_phase4() {

  GenCollectedHeap* gch = GenCollectedHeap::heap();
  Generation* pg = gch->perm_gen();
  
  pg->compact();
  
  GenCompactClosure blk;
  gch->generation_iterate(&blk, true);
  
  pg->post_compact(); // Shared spaces verification.
}
```
看到`compact`方法：
```
void CompactibleSpace::compact() {
  SCAN_AND_COMPACT(obj_size);
}
```
调用的是宏`SCAN_AND_COMPACT`，将对象复制到新地址。

#### 小结
**TenuredGeneration**执行GC分为4步：
* 递归标记GC Root存活对象
* 计算活跃对象在压缩完成之后的新地址
* 更新对象的引用地址
* 移动所有活跃对象到新地址






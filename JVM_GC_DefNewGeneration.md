---
layout: post
title: "HotSpot新生代DefNewGeneration的实现"
date: 2019-06-18 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文研究HotSpot新生代DefNewGeneration的实现。

<!--more-->

### DefNewGeneration的定义
```
class DefNewGeneration: public Generation {

  protected:
    Generation* _next_gen;
    int         _tenuring_threshold;   // Tenuring threshold for next collection.
    ageTable    _age_table;
    // Size of object to pretenure in words; command line provides bytes
    size_t        _pretenure_size_threshold_words;
    
    // True iff a promotion has failed in the current collection.
    bool   _promotion_failed;
    bool   promotion_failed() { return _promotion_failed; }
    
    
    /** 
    * 处理晋升失败（promotion failure） 
    *   如果在收集时，不能将一个对象从它所在的Eden区或from区拷贝出来，收集将失败。
    *   收集前，所有在新生代的 Root，都指向 Eden 或 From 区
    *
    *   收集失败后：
    *     # 对象A在 eden或from区，还有它的一个copy B在 to 区。这时，需要将 root指向A的指针改为指向B。
    *     # 所有新生代的对象都是 unmarked的。
    *     # Eden、from、to区域都需要 Full GC。
    */
    void handle_promotion_failure(oop);
    
    
    /**
    * 当 promotion failure 发生，删除“转发指针”(forwarding pointer)
    */
    void remove_forwarding_pointers();
    
    // ... 省略部分代码
    
    // Spaces
    EdenSpace*       _eden_space;
    ContiguousSpace* _from_space;
    ContiguousSpace* _to_space;
    
    // ...
}
```
总结一下DefNewGeneration定义的一些重要的属性和方法：
* eden、from、to区，还有对象的晋升阀值
* 获取space相关信息，如 capacity、used、free等
* 内存分配
* 对象遍历相关
* 晋升失败时的处理


### DefNewGeneration的初始化
```
DefNewGeneration::DefNewGeneration(ReservedSpace rs,
                                   size_t initial_size,
                                   int level,
                                   const char* policy)
  : Generation(rs, initial_size, level),
    _promo_failure_drain_in_progress(false),
    _should_allocate_from_space(false)
{

  MemRegion cmr((HeapWord*)_virtual_space.low(),
                (HeapWord*)_virtual_space.high());
  Universe::heap()->barrier_set()->resize_covered_region(cmr);

  // 1. 创建 Eden、from、to区
  _eden_space = new EdenSpace(this);
  _from_space = new ContiguousSpace();
  _to_space   = new ContiguousSpace();

  // 2. 如果_eden_space、_from_space、_to_space其中任何一个为空，
  //    说明新生代分配内存失败，则虚拟机退出
  if (_eden_space == NULL || _from_space == NULL || _to_space == NULL)
    vm_exit_during_initialization("Could not allocate a new gen space");
    
  // 3. 根据 SurvivorRatio 计算 survivor区的最大size
  //    默认是8，即Eden的size / survivor的size = 8， 
  //    即 Eden + from + to = 8 + 1 + 1 
  uintx alignment = GenCollectedHeap::heap()->collector_policy()->min_alignment();
  uintx size = _virtual_space.reserved_size();
  _max_survivor_size = compute_survivor_size(size, alignment);
  _max_eden_size = size - (2*_max_survivor_size);
  
  // ... 省略
}
```


### DefNewGeneration的GC过程
它的GC过程的实现在` DefNewGeneration::collect`方法。

#### 检查
```
void DefNewGeneration::collect(bool   full,
                               bool   clear_all_soft_refs,
                               size_t size,
                               bool   is_tlab) {
                               
  assert(full || size > 0, "otherwise we don't want to collect");
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  _next_gen = gch->next_gen(this);
  assert(_next_gen != NULL,
    "This must be the youngest gen, and not the only gen");
    
  // If the next generation is too full to accomodate promotion
  // from this generation, pass on collection; let the next generation
  // do it.
  if (!collection_attempt_is_safe()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print(" :: Collection attempt not safe :: ");
    }
    gch->set_incremental_collection_failed(); // Slight lie: we did not even attempt one
    return;
  }
  assert(to()->is_empty(), "Else not collection_attempt_is_safe"); 
  
  init_assuming_no_promotion_failure();
}
```
这里主要是一些检查：
* 确保这是一次 Full GC，或者需要分配的内存大小 size 大于0，否则不执行GC
* 由于这是新生代，确保它有下一个分代（老年代）
* 调用`collection_attempt_is_safe`方法，检查是否适合进行GC

其中，`collection_attempt_is_safe`方法如下：
```
bool DefNewGeneration::collection_attempt_is_safe() {
  if (!to()->is_empty()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print(" :: to is not empty :: ");
    }
    return false;
  }
  if (_next_gen == NULL) {
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    _next_gen = gch->next_gen(this);
    assert(_next_gen != NULL,
           "This must be the youngest gen, and not the only gen");
  }
  return _next_gen->promotion_attempt_is_safe(used());
}
```
这个方法根据以下条件判断GC是否可以执行：
* to区为空
* 下一个分代是否可以容纳新生代所有对象


#### 准备工作
```
  // These can be shared for all code paths
  IsAliveClosure is_alive(this);
  ScanWeakRefClosure scan_weak_ref(this);

  age_table()->clear();
  to()->clear(SpaceDecorator::Mangle);

  gch->rem_set()->prepare_for_younger_refs_iterate(false);

  assert(gch->no_allocs_since_save_marks(0),
         "save marks have not been newly set.");

  // Not very pretty.
  CollectorPolicy* cp = gch->collector_policy();

  FastScanClosure fsc_with_no_gc_barrier(this, false);
  FastScanClosure fsc_with_gc_barrier(this, true);

  set_promo_failure_scan_stack_closure(&fsc_with_no_gc_barrier);
  FastEvacuateFollowersClosure evacuate_followers(gch, _level, this,
                                                  &fsc_with_no_gc_barrier,
                                                  &fsc_with_gc_barrier);
```
这里的准备工作有：
* 初始化了几个的 XXClosure，它们的定义、作用后面再说。
* 清空age_table和to区
* 创建一个覆盖整个空间的数组GenRemSet，数组每个字节对应于堆的512字节，用于遍历新生代和老年代空间

#### 对根集对象标记
```
gch->gen_process_strong_roots(_level,
                                true,  // Process younger gens, if any,
                                       // as strong roots.
                                true,  // activate StrongRootsScope
                                false, // not collecting perm generation.
                                SharedHeap::SO_AllClasses,
                                &fsc_with_no_gc_barrier,
                                true,   // walk *all* scavengable nmethods
                                &fsc_with_gc_barrier);
```
对根集对象标记的过程在`GenCollectedHeap::gen_process_strong_roots`方法，其中，这个方法可以分为3部分：
* 处理当前分代
* 处理更低内存分代
* 处理更高内存分代

##### 处理当前分代
```
  if (!do_code_roots) {
    SharedHeap::process_strong_roots(activate_scope, collecting_perm_gen, so,
                                     not_older_gens, NULL, older_gens);
  } else {
    bool do_code_marking = (activate_scope || nmethod::oops_do_marking_is_active());
    CodeBlobToOopClosure code_roots(not_older_gens, /*do_marking=*/ do_code_marking);
    SharedHeap::process_strong_roots(activate_scope, collecting_perm_gen, so,
                                     not_older_gens, &code_roots, older_gens);
  }
```
看到其中的`SharedHeap::process_strong_roots`方法:
```
  StrongRootsScope srs(this, activate_scope);
  // General strong roots.
  assert(_strong_roots_parity != 0, "must have called prologue code");
  if (!_process_strong_tasks->is_task_claimed(SH_PS_Universe_oops_do)) {
    Universe::oops_do(roots);
    ReferenceProcessor::oops_do(roots);
    // Consider perm-gen discovered lists to be strong.
    perm_gen()->ref_processor()->weak_oops_do(roots);
  }
  // Global (strong) JNI handles
  if (!_process_strong_tasks->is_task_claimed(SH_PS_JNIHandles_oops_do))
    JNIHandles::oops_do(roots);
  // All threads execute this; the individual threads are task groups.
  if (ParallelGCThreads > 0) {
    Threads::possibly_parallel_oops_do(roots, code_roots);
  } else {
    Threads::oops_do(roots, code_roots);
  }
  
  // ... 省略
```
这个方法扫描了一定是GC Root的内存区域：
* Universe类中所引用的一些必须存活的对象 : `Universe::oops_do(roots)`
* 所有JNI Handles : `JNIHandles::oops_do(roots)`
* 所有线程的栈 : `Threads::oops_do(roots, code_roots)`
* 所有被Synchronize锁持有的对象 : `ObjectSynchronizer::oops_do(roots)`
* ...省略


##### 补充 -- HotSpot里面的 *-Closure
HotSpot VM里有很多以*-Closure方式命名的类。它们其实是封装起来的回调函数。为了让==GC的具体逻辑与对象内部遍历字段的逻辑能松耦合==，这部分都是通过回调函数来连接到一起的。

以上面代码的 `SharedHeap::process_strong_roots` 为例，它是处理根集合的对象的，而这自然包括遍历、处理2部分。

如上面的`Universe::oops_do(roots)`，`Universe`会让它的所有对象都调用传入的`OopClosure`方法。这样就做到遍历与具体逻辑松耦合。


说完了“遍历”，现在要说“处理逻辑”了。看到`FastScanClosure::do_oop_work`方法：
```
template <class T> inline void FastScanClosure::do_oop_work(T* p) {
  T heap_oop = oopDesc::load_heap_oop(p);
  // Should we copy the obj?
  if (!oopDesc::is_null(heap_oop)) {
    oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);
    if ((HeapWord*)obj < _boundary) {
      assert(!_g->to()->is_in_reserved(obj), "Scanning field twice?");
      oop new_obj = obj->is_forwarded() ? obj->forwardee()
                                        : _g->copy_to_survivor_space(obj);
      oopDesc::encode_store_heap_oop_not_null(p, new_obj);
      if (_gc_barrier) {
        // Now call parent closure
        do_barrier(p);
      }
    }
  }
}
```
上面代码的逻辑是：
* 如果这个指针指向的oop对象是没有标记过的，那么将这个oop对象拷贝到to区，并将新oop对象的地址放在老oop对象的markOop里面，表示这个对象已经复制到这个位置了。我们把这个指针称为“转发指针”（Forwarding pointer）。
* 修改指针的值，让它指向新的oop对象的地址
* 调用`do_barrier`方法，这是所谓的“写屏障”（write barrier）


##### 复制对象到To区的实现
```
oop DefNewGeneration::copy_to_survivor_space(oop old) {
  
  size_t s = old->size();
  oop obj = NULL;

  // Try allocating obj in to-space (unless too old)
  if (old->age() < tenuring_threshold()) {
    obj = (oop) to()->allocate(s);
  }

  // Otherwise try allocating obj tenured
  if (obj == NULL) {
    obj = _next_gen->promote(old, s);
    if (obj == NULL) {
      handle_promotion_failure(old);
      return old;
    }
  } else {
    // Prefetch beyond obj
    const intx interval = PrefetchCopyIntervalInBytes;
    Prefetch::write(obj, interval);

    // Copy obj
    Copy::aligned_disjoint_words((HeapWord*)old, (HeapWord*)obj, s);

    // Increment age if obj still in new generation
    obj->incr_age();
    age_table()->add(obj, s);
  }

  // Done, insert forward pointer to obj in this header
  old->forward_to(obj);

  return obj;
}
```
上面代码的逻辑：
* 判断该对象占用空间是否小于直接移动到老年代的阈值：
  * 是： 
    * 在to区分配内存，将原对象的数据复制到新对象。
    * 增加对象的复制计数和更新ageTable。
    * 设置原对象的对象头为转发指针(表示该对象已被复制，并指明该对象已经被复制到什么位置)
  * 否： 
    * 尝试在老年代分配内存（将这个对象晋升到老年代）
    * 如果晋升失败，调用`DefNewGeneration::handle_promotion_failure`方法处理，这个方法主要是把对象oop、markoop入栈，用于后面对象头的恢复
  

##### 补充 --  写屏障
HotSpot VM的分代式GC需要通过写屏障（write barrier）来维护一个记忆集合（remember set）——记录从old generation到young generation的跨代引用的数据结构。具体在代码中叫做CardTable。在minor GC时，old generation被remember set所记录下的区域会被看作根集合的一部分。而在minor GC过程中，每当有对象晋升到old generation都有可能产生新的跨代引用。



##### 处理更低的分代
<font color=red>这一小段代码在YGC时是不执行的，因为此时的“当前分代”就是新生代，它没有更年轻的分代了！</font>
```
  if (younger_gens_as_roots) {
    if (!_gen_process_strong_tasks->is_task_claimed(GCH_PS_younger_gens)) {
      for (int i = 0; i < level; i++) {
        not_older_gens->set_generation(_gens[i]);
        _gens[i]->oop_iterate(not_older_gens);
      }
      not_older_gens->reset_generation();
    }
  }
```
遍历所有比当前分代年轻的分代，调用`oop_iterate`方法：
```
void Generation::oop_iterate(OopClosure* cl) {
  GenerationOopIterateClosure blk(cl, _reserved);
  space_iterate(&blk);
}
```
通过使用`space_iterate`对分代中的各个Sapce进行遍历。`space_iterate`定义在类`Generation`，由各个具体分代实现这个方法。

DefNewGeneration的实现：
```
void DefNewGeneration::space_iterate(SpaceClosure* blk,
                                     bool usedOnly) {
  blk->do_space(eden());
  blk->do_space(from());
  blk->do_space(to());
}
```
分别对eden、from、to区进行处理。处理的逻辑是调用`SpaceClosure::do_space`方法：
```
virtual void do_space(Space* s) {
    s->oop_iterate(mr, cl);
}
```
因此，对于`DefNewGeneration`，就是分别调用eden、from、to 3 个Space（它们都是`ContiguousSpace`）的`oop_iterate`方法。

这里，Space的oop遍历在另一篇文章关于Space的实现里面会讲。

对于更低的分代，处理逻辑就是将存在于 **DefNewGeneration** 分代的对象，移动到To区。



##### 处理更高的分代
```
  for (int i = level+1; i < _n_gens; i++) {
    older_gens->set_generation(_gens[i]);
    rem_set()->younger_refs_iterate(_gens[i], older_gens);
    older_gens->reset_generation();
  }
```
* 这里的`rem_set()`是`GenRemSet`，这是用于记录老年代到新生代之间对象跨代引用的数据结构。
* 通过遍历这些oop对象，调用`FastScanClosure`

所以，对于更高的分代，也是把存在于**DefNewGeneration** 分代的对象，移动到To区。

##### 小结
对根集对象（GC Root）标记的过程，分为了3部分处理：
* 当前分代： 扫描一定是GC Root的内存区域。
* 更低分代： 扫描这些分代的每一个Space
* 更高分代： 根据`GenRemSet`数据结构，找到被跨代引用着的对象

处理逻辑都是通过`FastScanClosure`，将对象移动到To区域。现在，就完成了对根集对象的处理，将回收范围限制在**DefNewGeneration**内。



#### 对活跃对象标记
在上一步，已经将GC Root所直接引用的对象拷贝到To区了，这一步做的就是递归遍历这些对象所引用的对象到To区。
```
void DefNewGeneration::FastEvacuateFollowersClosure::do_void() {
  do {
    _gch->oop_since_save_marks_iterate(_level, _scan_cur_or_nonheap,
                                       _scan_older);
  } while (!_gch->no_allocs_since_save_marks(_level));
  guarantee(_gen->promo_failure_scan_is_complete(), "Failed to finish scan");
}
```

##### 循环条件
通过`no_allocs_since_save_marks`，分别检查当前分代和更高分代其scanned指针_saved_mark_word是否与当前空闲分配指针位置相同，即检查scanned指针是否追上空闲分配指针。

Space的相关指针如下：
```
|[ 已分配并且已扫描完的对象 ]|[ 已分配但未扫描完的对象 ]|[ 未分配空间 ]|
^                        ^                      ^             ^
bottom                   scanned                top           end
```

每次扫描一个对象，saved_mark_word（即scanned）会往前移动，期间也有新的对象会被拷贝到to-space，top也会往前移动，直到saved_mark_word追上top，说明to-space的对象都已经遍历完成。

因此，这里是循环判断各个内存代，是否有对象需要扫描，如有，调用`oop_since_save_marks_iterate`进行处理。否则，退出循环。

##### 处理
对每一个分代遍历：
```
void GenCollectedHeap::                                                 \
oop_since_save_marks_iterate(int level,                                 \
                             OopClosureType* cur,                       \
                             OopClosureType* older) {                   \
  _gens[level]->oop_since_save_marks_iterate##nv_suffix(cur);           \
  for (int i = level+1; i < n_gens(); i++) {                            \
    _gens[i]->oop_since_save_marks_iterate##nv_suffix(older);           \
  }                                                                     \
  perm_gen()->oop_since_save_marks_iterate##nv_suffix(older);           \
}
```
看到DefNewGeneration的实现：
```
void DefNewGeneration::                                         \
oop_since_save_marks_iterate##nv_suffix(OopClosureType* cl) {   \
  cl->set_generation(this);                                     \
  eden()->oop_since_save_marks_iterate##nv_suffix(cl);          \
  to()->oop_since_save_marks_iterate##nv_suffix(cl);            \
  from()->oop_since_save_marks_iterate##nv_suffix(cl);          \
  cl->reset_generation();                                       \
  save_marks();                                                 \
}
```
其实方法名字就说明了，“遍历处理从 scaned指针到top之间的对象”。

具体逻辑是分别调用了eden、from、to的`oop_since_save_marks_iterate`方法：
```
void ContiguousSpace::                                                    \
oop_since_save_marks_iterate##nv_suffix(OopClosureType* blk) {            \
  HeapWord* t;                                                            \
  HeapWord* p = saved_mark_word();                                        \
  assert(p != NULL, "expected saved mark");                               \
                                                                          \
  const intx interval = PrefetchScanIntervalInBytes;                      \
  do {                                                                    \
    t = top();                                                            \
    while (p < t) {                                                       \
      Prefetch::write(p, interval);                                       \
      debug_only(HeapWord* prev = p);                                     \
      oop m = oop(p);                                                     \
      p += m->oop_iterate(blk);                                           \
    }                                                                     \
  } while (t < top());                                                    \
                                                                          \
  set_saved_mark_word(p);                                                 \
}
```
这里，同样是调用`FastScanClosure`进行处理。

##### 小结
对活跃对象标记这一步，就是将对象（从上一步得到的对象所引用着的对象）移动到To区。
在这个过程中，需要遍历更高的内存代，因为对象在复制的过程中，是可能晋升的。


#### 处理引用 
先跳过.

#### 处理晋升成功
```
  if (!promotion_failed()) {
    // Swap the survivor spaces.
    eden()->clear(SpaceDecorator::Mangle);
    from()->clear(SpaceDecorator::Mangle);
    if (ZapUnusedHeapArea) {
      // This is now done here because of the piece-meal mangling which
      // can check for valid mangling at intermediate points in the
      // collection(s).  When a minor collection fails to collect
      // sufficient space resizing of the young generation can occur
      // an redistribute the spaces in the young generation.  Mangle
      // here so that unzapped regions don't get distributed to
      // other spaces.
      to()->mangle_unused_area();
    }
    swap_spaces();

    assert(to()->is_empty(), "to space should be empty now");

    // Set the desired survivor size to half the real survivor space
    _tenuring_threshold =
      age_table()->compute_tenuring_threshold(to()->capacity()/HeapWordSize);

    // A successful scavenge should restart the GC time limit count which is
    // for full GC's.
    AdaptiveSizePolicy* size_policy = gch->gen_policy()->size_policy();
    size_policy->reset_gc_overhead_limit_count();
    if (PrintGC && !PrintGCDetails) {
      gch->print_heap_change(gch_prev_used);
    }
    assert(!gch->incremental_collection_failed(), "Should be clear");
```
如果没有对象发生晋升失败，执行下面的逻辑：
* 既然所有对象都晋升成功了，说明存活对象都转移到了to区域或老年代，则通过clear方法清空eden和from区；
* 通过`swap_spaces`方法交换from和to区域。
* 还会将Eden的`_next_compaction_space`指向from，而from的为NUll。Full GC时会使用这个指针对Space压缩。即表示压缩eden区和from区（原To区）。


#### 处理晋升失败
```
  else {
    assert(_promo_failure_scan_stack.is_empty(), "post condition");
    _promo_failure_scan_stack.clear(true); // Clear cached segments.

    remove_forwarding_pointers();
    if (PrintGCDetails) {
      gclog_or_tty->print(" (promotion failed) ");
    }
    // Add to-space to the list of space to compact
    // when a promotion failure has occurred.  In that
    // case there can be live objects in to-space
    // as a result of a partial evacuation of eden
    // and from-space.
    swap_spaces();   // For uniformity wrt ParNewGeneration.
    from()->set_next_compaction_space(to());
    gch->set_incremental_collection_failed();

    // Inform the next generation that a promotion failure occurred.
    _next_gen->promotion_failure_occurred();

    // Reset the PromotionFailureALot counters.
    NOT_PRODUCT(Universe::heap()->reset_promotion_should_fail();)
  }
```
执行下面逻辑：
* 通过`remove_forwarding_pointers`恢复晋升失败对象的markOop。
* 将from和to区进行互换，还将设置to区设置为from区的下一个压缩区域。即表示eden、from、to都需要压缩。
* 设置新生代的收集失败标记
* 通知下一内存分代（老年代）发生了晋升失败

在对象发生晋升失败时，会把该对象的oop和markoop分别保存在`_objs_with_preserved_marks`和`_preserved_marks_of_objs`这2个栈。同时将对象头的“转发指针”指向自身（应该是为了避免重复遍历到这个对象）。

`remove_forwarding_pointers`做的就是从2个栈取出oop对象和对应的markOop，然后恢复对象头。

#### 总结
DefNewGeneration的YGC流程为：
1. 将GC Root直接引用的对象拷贝到To区
2. 遍历上一步的对象，拷贝这些对象所引用的对象（称为活跃对象）到To区
3. 还要处理自然晋升（对象年龄大于阀值导致的）和提前晋升（To区域空间不足）

这部分的代码分为7个部分：
* 检查
* 准备工作
* 对根集对象标记（GC Root），其中又细分为3部分：
  * 处理当前分代（扫描一定是GC Root的内存区域）
  * 处理更年轻的分代（由于现在是YGC，没有更年轻的分代）
  * 处理更年老的分代，借助`GenRemSet`，处理被老年代对象跨代引用着的对象
* 对活跃对象标记
  * 遍历每一个分代的每一个Space，直到没有未被扫描的对象
  * 每个Space根据指针`scanned`是否“追上”`top`可以知道有没有未被扫描的对象。
* 处理引用 (先跳过)
* 处理晋升成功
* 处理晋升失败





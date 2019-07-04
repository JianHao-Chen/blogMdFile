---
layout: post
title: "HotSpot堆的实现"
date: 2019-06-16 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文主要回顾Java GC方面的知识。

<!--more-->

### Java堆的实现
<img src="/assets/blogImg/JVM/GC/Heap/1.png" width=400>

#### CollectedHeap
CollectedHeap是一个抽象类，主要是定义Java堆必须实现的方法。以下是它的代码，省略了一些：
```
class CollectedHeap : public CHeapObj {

  protected:
    MemRegion _reserved;            // 表示一个内存区域，封装了start和size
    BarrierSet* _barrier_set;
    bool _is_gc_active;             // 当前是否STW
    int _n_par_threads;             // 当前执行GC任务的线程数量

    // 执行GC收集的次数
    unsigned int _total_collections;    
    unsigned int _total_full_collections;
    
    // Reason for current garbage collection.  Should be set to
    // a value reflecting no collection between collections.
    GCCause::Cause _gc_cause;
    GCCause::Cause _gc_lastcause;
    PerfStringVariable* _perf_gc_cause;
    PerfStringVariable* _perf_gc_lastcause;
    
    
    // Create a new tlab
    virtual HeapWord* allocate_new_tlab(size_t size);
    // Reinitialize tlabs before resuming mutators.
    virtual void resize_all_tlabs();
    
    // Allocate from the current thread's TLAB, with broken-out slow path.
    inline static HeapWord* allocate_from_tlab(Thread* thread, size_t size);
    static HeapWord* allocate_from_tlab_slow(Thread* thread, size_t size);
    
    // Allocate an uninitialized block of the given size, or returns NULL if
    // this is impossible.
    inline static HeapWord* common_mem_allocate_noinit(size_t size, bool is_noref, TRAPS);

    // Like allocate_init, but the block returned by a successful allocation
    // is guaranteed initialized to zeros.
    inline static HeapWord* common_mem_allocate_init(size_t size, bool is_noref, TRAPS);

    // Same as common_mem version, except memory is allocated in the permanent area
    // If there is no permanent area, revert to common_mem_allocate_noinit
    inline static HeapWord* common_permanent_mem_allocate_noinit(size_t size, TRAPS);

    // Same as common_mem version, except memory is allocated in the permanent area
    // If there is no permanent area, revert to common_mem_allocate_init
    inline static HeapWord* common_permanent_mem_allocate_init(size_t size, TRAPS);
    
    
  public:
    virtual jint initialize() = 0;
    virtual void post_initialize() = 0;
    
    MemRegion reserved_region() const { return _reserved; }
    address base() const { return (address)reserved_region().start(); }
  
    virtual size_t capacity() const = 0;
    virtual size_t used() const = 0;
  
    virtual size_t permanent_capacity() const = 0;
    virtual size_t permanent_used() const = 0;
    

    // Returns "TRUE" if "p" points into the reserved area of the heap.
    bool is_in_reserved(const void* p) const {
      return _reserved.contains(p);
    }
    
    // Returns "TRUE" if "p" points to the head of an allocated object in the
    // heap. Since this method can be expensive in general, we restrict its
    // use to assertion checking only.
    virtual bool is_in(const void* p) const = 0;
    
    virtual bool is_in_closed_subset(const void* p) const {
      return is_in_reserved(p);
    }
    
    virtual bool is_in_permanent(const void *p) const = 0;
    
    virtual bool is_permanent(const void *p) const = 0;
    
    inline static oop obj_allocate(KlassHandle klass, int size, TRAPS);
    inline static oop array_allocate(KlassHandle klass, int size, int length, TRAPS);
    inline static oop large_typearray_allocate(KlassHandle klass, int size, int length, TRAPS);
    
    
    inline static oop permanent_obj_allocate(KlassHandle klass, int size, TRAPS);
    // 分配oop，但不安装klass
    inline static oop permanent_obj_allocate_no_klass_install(KlassHandle klass,int size,TRAPS);
    // 用于安装klass
    inline static void post_allocation_install_obj_klass(KlassHandle klass,oop obj,int size);
    
    
    virtual HeapWord* mem_allocate(size_t size,
                                 bool is_noref,
                                 bool is_tlab,
                                 bool* gc_overhead_limit_was_exceeded) = 0;
    virtual HeapWord* permanent_mem_allocate(size_t size) = 0;
    
    
    virtual bool supports_tlab_allocation() const {
      return false;
    }
    virtual size_t tlab_capacity(Thread *thr) const {
      guarantee(false, "thread-local allocation buffers not supported");
      return 0;
    }
  
    // Perform a collection of the heap; intended for use in implementing
    // "System.gc".  This probably implies as full a collection as the
    // "CollectedHeap" supports.
    virtual void collect(GCCause::Cause cause) = 0;
    
     // This interface assumes that it's being called by the
    // vm thread. It collects the heap assuming that the
    // heap lock is already held and that we are executing in
    // the context of the vm thread.
    virtual void collect_as_vm_thread(GCCause::Cause cause) = 0;
    
    // Returns "true" iff there is a stop-world GC in progress.  (I assume
    // that it should answer "false" for the concurrent part of a concurrent
    // collector -- dld).
    bool is_gc_active() const { return _is_gc_active; }
    
    // Total number of GC collections (started)
    unsigned int total_collections() const { return _total_collections; }
    unsigned int total_full_collections() const { return      _total_full_collections;}
    
    // Iterate over all the ref-containing fields of all objects, calling
    // "cl.do_oop" on each. This includes objects in permanent memory.
    virtual void oop_iterate(OopClosure* cl) = 0;
    
    // Iterate over all objects, calling "cl.do_object" on each.
    // This includes objects in permanent memory.
    virtual void object_iterate(ObjectClosure* cl) = 0;
    
    
    // Similar to object_iterate() except iterates only
    // over live objects.
    virtual void safe_object_iterate(ObjectClosure* cl) = 0;
    
     // Behaves the same as oop_iterate, except only traverses
    // interior pointers contained in permanent memory. If there
    // is no permanent memory, does nothing.
    virtual void permanent_oop_iterate(OopClosure* cl) = 0    
    // Behaves the same as object_iterate, except only traverses
    // object contained in permanent memory. If there is no
    // permanent memory, does nothing.
    virtual void permanent_object_iterate(ObjectClosure* cl) = 0;
    
    // Print all GC threads (other than the VM thread)
    // used by this heap.
    virtual void print_gc_threads_on(outputStream* st) const = 0;
    void print_gc_threads() { print_gc_threads_on(tty); }
    // Iterator for all GC threads (other than VM thread)
    virtual void gc_threads_do(ThreadClosure* tc) const = 0;
}
```
总结上面代码，Java堆应有的主要功能有：
* 获取堆类型、容量、gc_cause、gc线程数这些信息
* 判断给定的指针是否在堆里（后面补充）
* oop对象分配
* 原生内存分配
* 内存回收
* 对堆内对象的遍历
* 打印堆用到的gc线程


### 堆初始化的过程


#### 堆初始化的入口
看到 `Universe::initialize_heap`方法：
```
jint Universe::initialize_heap() {

  if (UseParallelGC) {
#ifndef SERIALGC
    Universe::_collectedHeap = new ParallelScavengeHeap();
#else  // SERIALGC
    fatal("UseParallelGC not supported in java kernel vm.");
#endif // SERIALGC

  } else if (UseG1GC) {
#ifndef SERIALGC
    G1CollectorPolicy* g1p = new G1CollectorPolicy_BestRegionsFirst();
    G1CollectedHeap* g1h = new G1CollectedHeap(g1p);
    Universe::_collectedHeap = g1h;
#else  // SERIALGC
    fatal("UseG1GC not supported in java kernel vm.");
#endif // SERIALGC
  
  } else {
  
    GenCollectorPolicy *gc_policy;
    
    if (UseSerialGC) {
      gc_policy = new MarkSweepPolicy();
      
    } else if (UseConcMarkSweepGC) {
#ifndef SERIALGC
      if (UseAdaptiveSizePolicy) {
        gc_policy = new ASConcurrentMarkSweepPolicy();
      } else {
        gc_policy = new ConcurrentMarkSweepPolicy();
      }
#else   // SERIALGC
    fatal("UseConcMarkSweepGC not supported in java kernel vm.");
#endif // SERIALGC    
    
    } else { // default old generation
      gc_policy = new MarkSweepPolicy();
    }
  
    Universe::_collectedHeap = new GenCollectedHeap(gc_policy);
  }
  
  jint status = Universe::heap()->initialize();
  if (status != JNI_OK) {
    return status;
  }

  // ... 省略下面代码
}
```
上面代码的逻辑是：根据启动参数决定使用哪一种堆及收集策略。


#### HotSpot的收集策略实现

<img src="/assets/blogImg/JVM/GC/Heap/2.png" width=500>

#### MarkSweepPolicy的初始化过程

```
MarkSweepPolicy::MarkSweepPolicy() {
  initialize_all();
}
```
调用了`GenCollectorPolicy::initialize_all`方法：
```
virtual void initialize_all() {
    initialize_flags();
    initialize_size_info();
    initialize_generations();
}
```

##### initialize_flags
这个方法负责对新生代、老年代以及永久代的内存大小进行设置（包括对齐调整）。就是设置例如 NewSize、MaxNewSize、PermSize 这些变量的值。其中：

* 设置永久代的flag是在`CollectorPolicy::initialize_flags`实现的。
* 设置新生代的flag是在`GenCollectorPolicy::initialize_flags`实现的。
* 设置老年代的flag是在`TwoGenerationCollectorPolicy::initialize_flags`实现的。


##### initialize_size_info
设置新生代、老年代以及永久代的容量，包括初始值、最小值和最大值。
和`initialize_flags`相似：
* 设置永久代的size是在`CollectorPolicy::initialize_size_info`实现的。
* 设置新生代的size是在`GenCollectorPolicy::initialize_size_info`实现的。
* 设置老年代的size是在`TwoGenerationCollectorPolicy::initialize_size_info`实现的


##### initialize_generations
* HotSpot通过`GenerationSpec`、`PermanentGenerationSpec`，分别封装新生代（老年代）、永久代的细节，如名称、初始大小、最大大小等。
* 之所以使用这2个类去封装分代的如name、size这些信息，而不使用Generation的虚函数，是因为这些信息是分代初始化必须的。



#### GenCollectedHeap的初始化
看到`GenCollectedHeap::initialize`方法：
```
jint GenCollectedHeap::initialize() {
  CollectedHeap::pre_initialize();
  
  // 获取分代数量
  int i;
  _n_gens = gen_policy()->number_of_generations();

  // 对齐分代的初始值、最大值
  _gen_specs = gen_policy()->generations();
  PermanentGenerationSpec *perm_gen_spec =
                                collector_policy()->permanent_generation();
  // Make sure the sizes are all aligned.
  for (i = 0; i < _n_gens; i++) {
    _gen_specs[i]->align(alignment);
  }
  perm_gen_spec->align(alignment);
  
  // Allocate space for the heap.
  char* heap_address;
  size_t total_reserved = 0;
  int n_covered_regions = 0;
  ReservedSpace heap_rs(0);
  
  // 分配堆的地址，还没有分配内存！
  heap_address = allocate(alignment, perm_gen_spec, &total_reserved,
                          &n_covered_regions, &heap_rs);
                          
  _reserved = MemRegion((HeapWord*)heap_rs.base(),
                        (HeapWord*)(heap_rs.base() + heap_rs.size()));
  
  // It is important to do this in a way such that concurrent readers can't
  // temporarily think somethings in the heap.  (Seen this happen in asserts.)
  _reserved.set_word_size(0);
  _reserved.set_start((HeapWord*)heap_rs.base());
  size_t actual_heap_size = heap_rs.size() - perm_gen_spec->misc_data_size()
                                           - perm_gen_spec->misc_code_size();
  _reserved.set_end((HeapWord*)(heap_rs.base() + actual_heap_size));

  _rem_set = collector_policy()->create_rem_set(_reserved, n_covered_regions);
  set_barrier_set(rem_set()->bs());
  
  _gch = this;

  //  各个分代，通过 GenerationSpec::init 进行创建、初始化
  for (i = 0; i < _n_gens; i++) {
    ReservedSpace this_rs = heap_rs.first_part(_gen_specs[i]->max_size(),
                                              UseSharedSpaces, UseSharedSpaces);
    _gens[i] = _gen_specs[i]->init(this_rs, i, rem_set());
    heap_rs = heap_rs.last_part(_gen_specs[i]->max_size());
  }
  _perm_gen = perm_gen_spec->init(heap_rs, PermSize, rem_set());

  clear_incremental_collection_failed();
  
#ifndef SERIALGC
  // If we are running CMS, create the collector responsible
  // for collecting the CMS generations.
  if (collector_policy()->is_concurrent_mark_sweep_policy()) {
    bool success = create_cms_collector();
    if (!success) return JNI_ENOMEM;
  }
#endif // SERIALGC

  return JNI_OK;
}  
```
上面方法最重要的是对各个分代的创建和初始化，这是通过`GenerationSpec::init`方法完成：
```
Generation* GenerationSpec::init(ReservedSpace rs, int level,
                                 GenRemSet* remset) {
  switch (name()) {
    case Generation::DefNew:
      return new DefNewGeneration(rs, init_size(), level);
    
    case Generation::MarkSweepCompact:
      return new TenuredGeneration(rs, init_size(), level, remset);
      
#ifndef SERIALGC
    case Generation::ParNew:
      return new ParNewGeneration(rs, init_size(), level);

    case Generation::ASParNew:
      return new ASParNewGeneration(rs,
                                    init_size(),
                                    init_size() /* min size */,
                                    level);
    case Generation::ConcurrentMarkSweep: {
      assert(UseConcMarkSweepGC, "UseConcMarkSweepGC should be set");
      CardTableRS* ctrs = remset->as_CardTableRS();
      if (ctrs == NULL) {
        vm_exit_during_initialization("Rem set incompatibility.");
      }
      // Otherwise
      // The constructor creates the CMSCollector if needed,
      // else registers with an existing CMSCollector

      ConcurrentMarkSweepGeneration* g = NULL;
      g = new ConcurrentMarkSweepGeneration(rs,
                 init_size(), level, ctrs, UseCMSAdaptiveFreeLists,
                 (FreeBlockDictionary::DictionaryChoice)CMSDictionaryChoice);

      g->initialize_performance_counters();

      return g;
    }

    case Generation::ASConcurrentMarkSweep: {
      assert(UseConcMarkSweepGC, "UseConcMarkSweepGC should be set");
      CardTableRS* ctrs = remset->as_CardTableRS();
      if (ctrs == NULL) {
        vm_exit_during_initialization("Rem set incompatibility.");
      }
      // Otherwise
      // The constructor creates the CMSCollector if needed,
      // else registers with an existing CMSCollector

      ASConcurrentMarkSweepGeneration* g = NULL;
      g = new ASConcurrentMarkSweepGeneration(rs,
                 init_size(), level, ctrs, UseCMSAdaptiveFreeLists,
                 (FreeBlockDictionary::DictionaryChoice)CMSDictionaryChoice);

      g->initialize_performance_counters();

      return g;
    }
#endif // SERIALGC
     default:
      guarantee(false, "unrecognized GenerationName");
      return NULL;
  }
}                                 
```


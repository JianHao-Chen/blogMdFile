---
layout: post
title: "HotSpot栈的实现"
date: 2019-06-21 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文主要内容：
* 栈的基础知识
* 栈的实现：
  * 栈的结构
  * 方法调用的过程
  * 对栈帧内容读写

<!--more-->


### HotSpot的栈的基础
* 每启动一个Java线程，实际上是与内核线程一一对应的，因此HotSpot的栈是取决于系统实现。下面的分析都默认 x86,Linux 平台。
* 当线程调用一个Java方法时，JVM（具体看到`CallStub`相关）会在这个线程的栈中压入一个帧（称为当前帧）。
* 栈帧（stack frame）
  * 作用：支持虚拟机进行方法调用、方法执行
  * 包含：局部变量表、操作数栈、动态连接、方法返回地址等
* 对于活动线程，只有栈顶栈帧是有效的，执行引擎所运行的字节码指令都是只针对当前栈帧进行操作的。

#### 栈帧
* HotSpot没有区分Java方法帧和本地方法帧。
* Java方法帧可以是解释的，也可以是编译的。


#### 局部变量表
* 定义：是一组变量值存储空间，用于存放方法参数和方法内部定义的局部变量。
* 特点：
  * 以变量槽(Slot)为最小单位--32位虚拟机的一个Slot可以存放一个32位的数据类型。
  * 局部变量表的容量，由Class文件的方法的Code属性的 max_locals 决定。
  * 对于实例方法（非static）的调用传参，第0个Slot用于传递方法所属对象实例的引用，即Java方法里面使用的 this。
  * Slot可以重用，当Slot中的变量超出作用域，下一次分配Slot时，会覆盖原数据。
  * Slot对对象的引用，会影响到GC。
  * 不会为局部变量赋初值。
  

#### 操作数栈
* 定义：Java虚拟机的解释执行引擎被称为基于栈的执行引擎，其中的栈就是操作数栈。操作数栈又叫操作栈，或表达式栈（expression stack）。
* 特点：
  * 操作数栈的深度在编译期决定（Code属性的 max_stacks）。
  * 它作为虚拟机计算时临时数据的存储区域

#### 其他
除了局部变量表、操作数栈之外，栈帧还包含如下数据：
* 指向常量池的指针
* 返回值（将返回值压入发起调用方法的栈）
* 对方法异常表的引用


### HotSpot栈帧的实现

HotSpot的栈帧相关类图如下：

<img src="/assets/blogImg/JVM/Stack/1.png" width=400>

#### 涉及的类的介绍
#####  vframe
```
class vframe: public ResourceObj {
  protected:
    frame        _fr;      // Raw frame behind the virtual frame.
    RegisterMap  _reg_map; // Register map for the raw frame (used to handle callee-saved registers).
    JavaThread*  _thread;  // The thread owning the raw frame.
  public:
    // Factory method for creating vframes
    static vframe* new_vframe(const frame* f, const RegisterMap *reg_map, JavaThread* thread);    
    
    // Accessors
  frame              fr()           const { return _fr;       }
  CodeBlob*          cb()         const { return _fr.cb();  }
  nmethod*           nm()         const {
      assert( cb() != NULL && cb()->is_nmethod(), "usage");
      return (nmethod*) cb();
  }
  
  // Returns the sender vframe
  virtual vframe* sender() const;
  
  // Returns the next javaVFrame on the stack (skipping all other kinds of frame)
  javaVFrame *java_sender() const;
  
  // Answers if the this is the top vframe in the frame, i.e., if the sender vframe
  // is in the caller frame
  virtual bool is_top() const { return true; }

  // Returns top vframe within same frame (see is_top())
  virtual vframe* top() const;
}
```
vframe表示的是虚拟机层面的栈帧，它的作用是：
* 定义了相关的成员变量、方法
* 封装了`frame`（这个是物理栈帧在HotSpot的表示）


##### javaVFrame
```
class javaVFrame: public vframe {
  public:
    // JVM state
    virtual methodOop                    method()         const = 0;
    virtual int                          bci()            const = 0;
    virtual StackValueCollection*        locals()         const = 0;
    virtual StackValueCollection*        expressions()    const = 0;
    // the order returned by monitors() is from oldest -> youngest#4418568
    virtual GrowableArray<MonitorInfo*>* monitors()       const = 0;
    // ... 省略
}
```
javaVFrame表示的是Java层面的栈帧。

##### frame
**frame**的定义如下，省略部分代码：
```
class frame VALUE_OBJ_CLASS_SPEC {
  private:
    intptr_t* _sp; // stack pointer (from Thread::last_Java_sp)
    address   _pc; // program counter (the next instruction after the call)
    
    CodeBlob* _cb; // CodeBlob that "owns" pc
 
  public:
  
    // returns the frame size in stack slots
    int frame_size(RegisterMap* map) const;
    
    // returns the sending frame
    frame sender(RegisterMap* map) const;
  
  
    // All frames:
    
    // A low-level interface for vframes:
    
    intptr_t* addr_at(int index) const             { return &fp()[index];    }
    intptr_t  at(int index) const                  { return *addr_at(index); }
    
     // accessors for locals
    oop obj_at(int offset) const                   { return *obj_at_addr(offset);  }
    void obj_at_put(int offset, oop value)         { *obj_at_addr(offset) = value; }
  
    // Return address
    address  sender_pc() const;
    
    // returns the stack pointer of the calling frame
    intptr_t* sender_sp() const;
    
    
    /**
    * ----------- Interpreter frames:  
    */
    
    // Locals
    // The _at version returns a pointer because the address is used for GC.
    intptr_t* interpreter_frame_local_at(int index) const;

    void interpreter_frame_set_locals(intptr_t* locs);
    
    // byte code index
    jint interpreter_frame_bci() const;
    void interpreter_frame_set_bci(jint bci);
    
    // byte code pointer
    address interpreter_frame_bcp() const;
    void    interpreter_frame_set_bcp(address bcp);
    
    // method data pointer
    address interpreter_frame_mdp() const;
    void    interpreter_frame_set_mdp(address dp);
    
    // expression stack (may go up or down, direction == 1 or -1)
    intptr_t* interpreter_frame_expression_stack() const;
    static  jint  interpreter_frame_expression_stack_direction();
    
    // The _at version returns a pointer because the address is used for GC.
    intptr_t* interpreter_frame_expression_stack_at(jint offset) const;
    
    // top of expression stack
    intptr_t* interpreter_frame_tos_at(jint offset) const;
    intptr_t* interpreter_frame_tos_address() const;
    
    jint  interpreter_frame_expression_stack_size() const;
    intptr_t* interpreter_frame_sender_sp() const;
    

    // BasicObjectLocks:
    // ... 省略
    
    // Method & constant pool cache
    methodOop interpreter_frame_method() const;
    void interpreter_frame_set_method(methodOop method);
    methodOop* interpreter_frame_method_addr() const;
    constantPoolCacheOop* interpreter_frame_cache_addr() const;
  
    // Entry frames
    JavaCallWrapper* entry_frame_call_wrapper() const;
    intptr_t* entry_frame_argument_at(int offset) const;
    
    
    /**
    * ----------- Compiled frames:  
    */
    // ... 省略
    

#ifdef TARGET_ARCH_x86
# include "frame_x86.hpp"
#endif
#ifdef TARGET_ARCH_sparc
# include "frame_sparc.hpp"
#endif
#ifdef TARGET_ARCH_zero
# include "frame_zero.hpp"
#endif
#ifdef TARGET_ARCH_arm
# include "frame_arm.hpp"
#endif
#ifdef TARGET_ARCH_ppc
# include "frame_ppc.hpp"
#endif    
}
```
由于栈帧是基于系统实现的，所以可以看到`frame`类通过包含头文件的方式，将平台相关的代码引入。

而`frame`类定义了栈帧中平台无关的代码：
* 成员变量如：栈指针、返回地址
* 方法如：
  * 读写局部变量表
  * 获取返回地址
  * 获取调用当前方法的方法的栈指针
  * 读写字节码索引/指针
  * 获取操作数栈的大小、顶部元素
  * 偏向锁相关


##### frame_x86
```
public:

  enum {
    pc_return_offset                                 =  0,
    // All frames
    link_offset                                      =  0,
    return_addr_offset                               =  1,
    // non-interpreter frames
    sender_sp_offset                                 =  2,
    
#ifndef CC_INTERP

    // Interpreter frames
    interpreter_frame_result_handler_offset          =  3, // for native calls only
    interpreter_frame_oop_temp_offset                =  2, // for native calls only

    interpreter_frame_sender_sp_offset               = -1,
    // outgoing sp before a call to an invoked method
    interpreter_frame_last_sp_offset                 = interpreter_frame_sender_sp_offset - 1,
    interpreter_frame_method_offset                  = interpreter_frame_last_sp_offset - 1,
    interpreter_frame_mdx_offset                     = interpreter_frame_method_offset - 1,
    interpreter_frame_cache_offset                   = interpreter_frame_mdx_offset - 1,
    interpreter_frame_locals_offset                  = interpreter_frame_cache_offset - 1,
    interpreter_frame_bcx_offset                     = interpreter_frame_locals_offset - 1,
    interpreter_frame_initial_sp_offset              = interpreter_frame_bcx_offset - 1,

    interpreter_frame_monitor_block_top_offset       = interpreter_frame_initial_sp_offset,
    interpreter_frame_monitor_block_bottom_offset    = interpreter_frame_initial_sp_offset,
#endif // CC_INTERP

  // ... 省略
};

private:
  // an additional field beyond _sp and _pc:
  intptr_t*   _fp; // frame pointer
  
  intptr_t* ptr_at_addr(int offset) const {
    return (intptr_t*) addr_at(offset);
  }
  
  // Constructors
  frame(intptr_t* sp, intptr_t* fp, address pc);
  
  // accessors for the instance variables
  intptr_t*   fp() const { return _fp; }
  
  inline address* sender_pc_addr() const;
  
  // ... 省略
```
这里定义了x86平台下栈帧的实现：
* 定义的enum描述的是栈帧内各个元素的相对位置
* 定义了成员变量：帧指针



#### Java方法的调用过程
在HotSpot，JVM调用Java函数是通过 **JavaCalls** 模块完成的：由 **JavaCalls** 模块创建栈帧，保存调用者的帧指针。



##### JavaCalls
JavaCalls 按照调用类型的不同实现了多个调用接口：call_special、call_virtual、call_static，它们都调用低层接口 call。
```
void JavaCalls::call(JavaValue* result, methodHandle method, JavaCallArguments* args, TRAPS) {
  // Check if we need to wrap a potential OS exception handler around thread
  // This is used for e.g. Win32 structured exception handlers
  assert(THREAD->is_Java_thread(), "only JavaThreads can make JavaCalls");
  // Need to wrap each and everytime, since there might be native code down the
  // stack that has installed its own exception handlers
  os::os_exception_wrapper(call_helper, result, &method, args, THREAD);
}
```
低层接口 call 又通过`call_helper`函数（封装异常处理的函数）完成方法的调用：
```
// do call
{ JavaCallWrapper link(method, receiver, result, CHECK);
  { HandleMark hm(thread);  // HandleMark used by HandleMarkCleaner

    StubRoutines::call_stub()(
      (address)&link,
      // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
      result_val_address,          // see NOTE above (compiler problem)
      result_type,
      method(),
      entry_point,
      args->parameters(),
      args->size_of_parameters(),
      CHECK
    );

    result = link.result();  // circumvent MS C++ 5.0 compiler bug (result is clobbered across call)
    // Preserve oop return value across possible gc points
    if (oop_result_flag) {
      thread->set_vm_result((oop) result->get_jobject());
    }
  }
} // Exit JavaCallWrapper (can block - potential return oop must bereserved)
```
在讲解上面的代码之前，需要先补充一些知识。

##### StubRountines

**Rountine** 是在HotSpot里面被共享并频繁调用的例程，每一个例程就是虚拟机中某些基本操作的可以被单独调用的程序片段。

上面代码的`call_stub`就是其中一个例程，用于从C调用Java代码。

而这些例程会在JVM启动时初始化，简单描述流程是：
1. 入口是`init_globals()`方法，它分别调用`stubRoutines_init1()`和`stubRoutines_init2()`方法，这2个方法的区别是一个仅生成初始化例程，另一个生成所有例程。
2. 以`stubRoutines_init1()`为例，它调用了`StubGenerator`的`generate_initial`方法,而我们关心的 **call_stub** 就在这里被创建。

实际上，这些例程是平台相关的，我们接下来分析的 **call_stub**，它创建的函数是在 *stubGenerator_x86_32.cpp* 里面。

##### call_stub
```
address generate_call_stub(address& return_address) {

  // （1） 记录 PC 寄存器
  address start = __ pc();
  
  // （2）创建局部变量保存信息
  const Address rsp_after_call(rbp, -4 * wordSize);     // 上一栈帧基址
  const int     locals_count_in_bytes  (4*wordSize);    // 
  const Address mxcsr_save    (rbp, -4 * wordSize);     // 保存mxcsr的值
  const Address saved_rbx     (rbp, -3 * wordSize);     // 保存rbx的值
  const Address saved_rsi     (rbp, -2 * wordSize);     // 保存rsi的值
  const Address saved_rdi     (rbp, -1 * wordSize);     // 保存rdi的值
  const Address result        (rbp,  3 * wordSize);     // Java方法的返回值
  const Address result_type   (rbp,  4 * wordSize);     // 返回值的类型
  const Address method        (rbp,  5 * wordSize);     // Java方法的Oop
  const Address entry_point   (rbp,  6 * wordSize);     // Java方法的入口
  const Address parameters    (rbp,  7 * wordSize);     // 调用Java方法的参数
  const Address parameter_size(rbp,  8 * wordSize);     // 参数的数目
  const Address thread        (rbp,  9 * wordSize);     // 线程
  
  // （3）进入新的栈帧，通过2条指令：  push(rbp);  mov(rbp, rsp);
  __ enter();         
  
  // （4）计算为参数和寄存器保存需要的空间，并移动栈指针 rsp跳过这段预留空间
  __ movptr(rcx, parameter_size);              // parameter counter
  __ shlptr(rcx, Interpreter::logStackElementSize); // convert parameter count to bytes
  __ addptr(rcx, locals_count_in_bytes);       // reserve space for register saves
  __ subptr(rsp, rcx);
  __ andptr(rsp, -(StackAlignmentInBytes));    // Align stack
  
  // （5）保存寄存器 rdi, rsi, rbx, 这3个是被调用者保存寄存器，所以由本方法保存
  __ movptr(saved_rdi, rdi);
  __ movptr(saved_rsi, rsi);
  __ movptr(saved_rbx, rbx);
  
  // （6）按参数出现顺序复制参数
  //  ... 省略
  
  // （7）调用Java方法：
  //  将 Java方法的入口地址放到 rax寄存器，然后调用 call(rax)完成方法跳转
  __ BIND(parameters_done);
  __ movptr(rbx, method);           // get methodOop
  __ movptr(rax, entry_point);      // get entry_point
  __ mov(rsi, rsp);                 // set sender sp
  BLOCK_COMMENT("call Java function");
  __ call(rax);
  
  //（8）保存返回结果
  // 如果返回结果的类型是 long、float、double，则跳转到相应的 label
  __ movptr(rdi, result);
  Label is_long, is_float, is_double, exit;
  __ movl(rsi, result_type);
  __ cmpl(rsi, T_LONG);
  __ jcc(Assembler::equal, is_long);
  __ cmpl(rsi, T_FLOAT);
  __ jcc(Assembler::equal, is_float);
  __ cmpl(rsi, T_DOUBLE);
  __ jcc(Assembler::equal, is_double);
  
  // （8.1）handle T_INT case（返回结果保存在 rax）
  __ movl(Address(rdi, 0), rax);
  // （8.2）相当于设置了一个 exit 的 lable
  __ BIND(exit);
  
  // （9）弹出参数
  //  rsp_after_call 保存的是上一栈帧的基址， lea 指令将这个值赋给 rsp
  __ lea(rsp, rsp_after_call);
  
  // （10）恢复 rdi， rsi ，rbx 寄存器
  __ movptr(rbx, saved_rbx);
  __ movptr(rsi, saved_rsi);
  __ movptr(rdi, saved_rdi);
  __ addptr(rsp, 4*wordSize);
  
  
  // (11) 恢复上一栈帧：之前 push 了 rbp（上一栈帧），
  __ pop(rbp);
  
  // （12）返回
  // ret 指令作用：从当前 %rsp 指向的位置（即栈顶）弹出数据，并跳转到此数据代表的地址处
  // 此时栈顶数据是本方法调用后的返回地址。
  __ ret(0);

  
  // （13）对Java方法返回结果类型为 long 、float、double的处理
  //  ... 省略
  
  // 返回 start，作为这个 call_stub的地址，提供给其他模块调用
  return start;
}
```
##### <font color=red> **补充**</font>

（1）这个例程在创建变量保存信息时，那保存的是什么呢？

就是上面代码的这一句：
```
StubRoutines::call_stub()(
      (address)&link,
      // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
      result_val_address,          // see NOTE above (compiler problem)
      result_type,
      method(),
      entry_point,
      args->parameters(),
      args->size_of_parameters(),
      CHECK
    );
```
就是`call_helper`函数调用`call_stub`时传递的这8个参数！！！


（2） 画出上面CallStub的栈帧示意图：

<img src="/assets/blogImg/JVM/Stack/2.png" width=400>

##### 方法入口（method entry point，这部分与解释器有关）

* 真正执行Java字节码的是**解释器**。
* JVM启动时，**代码生成器**会为字节码、JVM内部例程生成**Codelets**，并存储在 **CodeCache** 中，以供解释器使用。
* 解释器将 JVM执行Java方法、native方法等方法调用所需要的共同工作提取出来，封装成 **方法入口**，作为 **Codelets** 的一部分。
* JVM调用的方法的类型不同，对应的栈帧也不同。
* 在`AbstractInterpreter`里面，定义了枚举类型`MethodKind`作为Hotspot内部方法类型。
* 这些方法入口就在`TemplateInterpreterGenerator::generate_all()`里面创建的。


##### zerolocals 方法的栈帧结构
在HotSpot的方法类型中，***zerolocals*** 表示需要初始化局部变量的方法。

通过`AbstractInterpreterGenerator::generate_method_entry`方法，JVM根据方法的类型决定生成哪一种方法入口。

而***zerolocals***的方法入口是通过`generate_normal_entry`方法生成的。我们通过这个方法，可以看出一般的Java方法执行前的栈帧结构。

###### 解释器栈帧结构--CallStub方法（简化）

<img src="/assets/blogImg/JVM/Stack/3.png" width=400>

开始分析`generate_normal_entry`方法：
```
address InterpreterGenerator::generate_normal_entry(bool synchronized) {
  // determine code generation flags
  bool inc_counter  = UseCompiler || CountCompiledCalls;
  
  // rbx,: methodOop
  // rsi: sender sp
  // methodOop的指针保存在 rbx， sender sp 保存在 rsi，参数size保存在 rcx（这是由 CallStub做的）
  address entry_point = __ pc();


  const Address size_of_parameters(rbx, methodOopDesc::size_of_parameters_offset());
  const Address size_of_locals    (rbx, methodOopDesc::size_of_locals_offset());
  const Address invocation_counter(rbx, methodOopDesc::invocation_counter_offset() + InvocationCounter::counter_offset());
  const Address access_flags      (rbx, methodOopDesc::access_flags_offset());

  // get parameter size (always needed)
  __ load_unsigned_short(rcx, size_of_parameters);
  
  // (1) 计算局部变量的大小，存储在 rdx
  __ load_unsigned_short(rdx, size_of_locals);       // get size of locals in words
  __ subl(rdx, rcx);                                // rdx = no. of additional locals
  
  // (2) 对栈空间大小进行检查，判断是否会发生栈溢出，具体实现不展开。
  generate_stack_overflow_check();
  
  
  // (3) 栈顶元素出栈，获取返回地址，保存在rax中
  __ pop(rax);
  
  // (4) 计算第一个参数的地址，存放在 (rdi)
  // 计算逻辑： rsp + rcx * 8 - 8 (假设 wordSize == 8)
  __ lea(rdi, Address(rsp, rcx, Interpreter::stackElementScale(), -wordSize));
  
  // (5) 为局部变量分配栈空间
  // rdx - # of additional locals
  // allocate space for locals
  // explicitly initialize locals
  {
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit);               // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int32_t)NULL_WORD);                      // initialize local variables
    __ decrement(rdx);                                // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }
```

到此，栈帧的结构为：

###### 解释器栈帧结构-- 创建方法入口-- 为局部变量分配空间

<img src="/assets/blogImg/JVM/Stack/4.png" width=400>

```

  // (6) 将方法的调用次数保存在rcx中
  if (inc_counter) {
    __ movl(rcx, invocation_counter);
  }
  
  // （7）初始化栈帧的固定部分
  // initialize fixed part of activation frame
  generate_fixed_frame(false);
```
有必要进入`generate_fixed_frame`方法看看：

```
// Generate a fixed interpreter frame. This is identical setup for
// interpreted methods and for native methods hence the shared code.
//
// Args:
//      rax: return address
//      rbx: methodOop
//      r14: pointer to locals
//      r13: sender sp
//      rdx: cp cache
void TemplateInterpreterGenerator::generate_fixed_frame(bool native_call) {

  __ push(rax);        // save return address
  __ enter();          // save old & set new rbp
  __ push(r13);        // set sender sp
  __ push((int)NULL_WORD); // leave last_sp as null
  __ movptr(r13, Address(rbx, methodOopDesc::const_offset()));      // get constMethodOop
  __ lea(r13, Address(r13, constMethodOopDesc::codes_offset())); // get codebase
  __ push(rbx);        // save methodOop
  
  __ push(0);
  
  __ movptr(rdx, Address(rbx, methodOopDesc::const_offset()));
  __ movptr(rdx, Address(rdx, constMethodOopDesc::constants_offset()));
  __ movptr(rdx, Address(rdx, constantPoolOopDesc::cache_offset_in_bytes()));
  __ push(rdx); // set constant pool cache
  __ push(r14); // set locals pointer
  
  if (native_call) {
    __ push(0); // no bcp
  } else {
    __ push(r13); // set bcp
  }
  __ push(0); // reserve word for pointer to expression stack bottom
  __ movptr(Address(rsp, 0), rsp); // set expression stack bottom
}
```
简单来说，这个方法做的是为解释执行的Java方法准备栈帧，此时栈帧结构如下：

###### 解释器栈帧结构-- 创建Java栈帧

<img src="/assets/blogImg/JVM/Stack/5.png" width=400>

继续看回`generate_normal_entry`方法：
```
  // （8）增加方法的调用计数并且检查是否发生栈溢出
  Label invocation_counter_overflow;
  Label profile_method;
  Label profile_method_continue;
  if (inc_counter) {
    generate_counter_incr(&invocation_counter_overflow, &profile_method, &profile_method_continue);
  }
  
  // （9）同步方法的Monitor对象分配和方法的加锁
  if (synchronized) {
    // Allocate monitor and lock method
    lock_method();
  } else {
  }
  
  // ...
  
  return entry_point;
}  
```



#### 对x86栈帧的内容的访问
上面已经给出了一般Java方法(zerolocals方法)的解释器帧的内存布局。

如上图所示，JVM对帧内元素的读写是按照“ rbp + offset”的形式进行的。

举一个例子，Java Thread获取栈帧的局部变量的过程：

```
intptr_t* frame::interpreter_frame_local_at(int index) const {
  const int n = Interpreter::local_offset_in_bytes(index)/wordSize;
  return &((*interpreter_frame_locals_addr())[n]);
}
```
通过`interpreter_frame_locals_addr()`得到局部变量表的基址，加上偏移量得到。

继续看如何得到局部变量表的基址：
```
// frame_x86.inline.hpp
inline intptr_t** frame::interpreter_frame_locals_addr() const {
  return (intptr_t**)addr_at(interpreter_frame_locals_offset);
}
```
继续看`addr_at`的实现（在frame.hpp）：
```
intptr_t* addr_at(int index) const             { return &fp()[index];    }
```
再看`fp()`的实现：
```
// frame_x86.hpp
intptr_t*   fp() const { return _fp; }
```
而`_fp`就是栈帧。


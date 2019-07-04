---
layout: post
title: "HotSpot的类加载--Linking"
date: 2019-06-10 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---





在学习HotSpot的类加载部分源码的过程中，写下以下几篇文章：
* HotSpot的类加载--概述
* HotSpot的类加载--Loading
* HotSpot的类加载--Linking（本文）
* HotSpot的类加载--Initialization

<!--more-->

### 链接（Linking）

链接的实现：
```
bool instanceKlass::link_class_impl(
    instanceKlassHandle this_oop, bool throw_verifyerror, TRAPS) {

    
    // 【1】link super class before linking this class
    instanceKlassHandle super(THREAD, this_oop->super());
    if (super.not_null()) {
        link_class_impl(super, throw_verifyerror, CHECK_false);
    }
    
    // 【2】link all interfaces implemented by this class before linking this class
    objArrayHandle interfaces (THREAD, this_oop->local_interfaces());
    int num_interfaces = interfaces->length();
    for (int index = 0; index < num_interfaces; index++) {
        HandleMark hm(THREAD);
        instanceKlassHandle ih(THREAD, klassOop(interfaces->obj_at(index)));
        link_class_impl(ih, throw_verifyerror, CHECK_false);
    }
    
    // in case the class is linked in the process of linking its superclasses
    if (this_oop->is_linked()) {
        return true;
    }
    
    
    // 【3】verification & rewriting
    {
        // 加锁
        ObjectLocker ol(this_oop, THREAD);
         
        // rewritten will have been set if loader constraint error found
        // on an earlier link attempt
        // don't verify or rewrite if already rewritten
        if (!this_oop->is_linked()) {
        
            if (!this_oop->is_rewritten()) {
                bool verify_ok = verify_code(this_oop, throw_verifyerror, THREAD);
                if (!verify_ok) {
                    return false;
                }
                
                // Just in case a side-effect of verify linked this class already
                // (which can sometimes happen since the verifier loads classes
                // using custom class loaders, which are free to initialize things)
                if (this_oop->is_linked()) {
                    return true;
                }
            
                // also sets rewritten
                // “重写”：是为了支持更好的解析器运行性能，向常量池添加缓存，
                // 并调整相应字节码的常量池索引重新指向常量池Cache索引
                this_oop->rewrite_class(CHECK_false);
            }
            
            
            // relocate jsrs and link methods after they are all rewritten
            // 为Java方法配置编译器或解析器入口(entry point)
            this_oop->relocate_and_link_methods(CHECK_false);
            
            // Initialize the vtable and interface table after
            // methods have been rewritten since rewrite may
            // fabricate new methodOops.
            // also does loader constraint checking
            if (!this_oop()->is_shared()) {
                ResourceMark rm(THREAD);
                this_oop->vtable()->initialize_vtable(true, CHECK_false);
                this_oop->itable()->initialize_itable(true, CHECK_false);
            }
            
            this_oop->set_init_state(linked);
            if (JvmtiExport::should_post_class_prepare()) {
                Thread *thread = THREAD;
                assert(thread->is_Java_thread(), "thread->is_Java_thread()");
                JvmtiExport::post_class_prepare((JavaThread *) thread, this_oop());
            }
        }
    }
    return true;
}    
```

#### 验证

##### 目的
确保字节流包含的信息是安全的。

虽然Java编译器可以检查到如错误类型转换等错误，但是，Class文件并不一定由编译器编译得到。因此，**验证**是必须的。


##### 步骤
从JVM规范来看，主要有4个步骤：==文件格式验证、元数据验证、字节码验证、符号引用验证==。

##### 文件格式验证
这里验证的有：
* magic number是否为 0xCAFEBABE
* 主次版本号是否在本版本虚拟机处理范围内
* 对常量池的检验：
  * 常量池大小
  * 严格按照规范读取常量池，如果遇到格式不对、不支持的tag等就报错
* 对
* ... 省略


###### 实现
HotSpot对此的实现，是在“加载”（Loading）阶段的，在`parseClassFile`方法：
```
// Magic value
u4 magic = cfs->get_u4_fast();
guarantee_property(magic == JAVA_CLASSFILE_MAGIC,
                     "Incompatible magic value %u in class file %s",
                     magic, CHECK_(nullHandle));
```
上面是读取字节流并检验magic number的代码。


###### 小结
* 这个阶段的验证是基于字节流的，只有通过验证，才能进入内存的方法区（就是在方法区创建`klassOop`，把从流中读取到信息赋值到它里面）
* ==这里的验证，关注的是文件格式方面的==


##### 元数据验证
验证的内容有：
* 这个类能否访问它的父类、它所实现的接口
* 这个类有没有重写了父类的`final`方法
* ....

###### 实现
这部分的检查，有一些其实在读取流的时候就做了（如是否继承了`final`类），有些是在创建好`klassOop`之后的，<font color='red'>但都在`parseClassFile`方法里面！</font>

###### 小结
==这里的验证，是对类的元数据方面的，关注的是有没有违反Java语言规范==。


##### 字节码验证

###### 目的
通过数据流和控制流分析，确定程序语意是合法的。

###### 途径
分析、检验方法体，保证没有“非法”的操作，例如：
* 保证操作数栈的数据类型和字节码能配合工作
* 保证跳转指令不会跳转到方法体以外的字节码指令上
* 保证方法体的类型转换是合法的
* ....


###### 实现
由于这部分非常复杂，HotSpot对此做了优化：
* JDK 1.6之后给方法体的Code属性的属性表上加上`StackMapTable`属性，这项属性描述了方法体的所有基本块（Basic Block，按照控制流拆分的代码块）开始时本地变量表和操作数栈应有的状态。于是，在字节码验证期间，就不需要根据程序推导这些状态的合法性，只需要检查`StackMapTable`属性中的记录即可。总结为一句话：将字节码的检验从推导变为类型检查。
* 可以使用 `-XX:-UseSplitVerify` 选项来关闭这个优化，或者使用参数 `-XX:+FailOverToOldVerify`在类型检查失败时，退回到类型推导进行检验。
* 在JDK 1.7 之后，对于主版本 > 50 的 class文件，强制使用类型检查，不运行使用类型推导。

相关代码看到`instanceKlass::verify_code`，它调用了`Verifier`模块的`verify`方法：
```
bool Verifier::verify(instanceKlassHandle klass, Verifier::Mode mode, bool should_verify_class, TRAPS) {

    // If the class should be verified, first see if we can use the split
    // verifier.  If not, or if verification fails and FailOverToOldVerifier
    // is set, then call the inference verifier.
    if (is_eligible_for_verification(klass, should_verify_class)) {
     
        if (UseSplitVerifier &&
            klass->major_version() >= STACKMAP_ATTRIBUTE_MAJOR_VERSION) {
            
            ClassVerifier split_verifier(klass, message_buffer, message_buffer_len, THREAD);
            split_verifier.verify_class(THREAD);
            exception_name = split_verifier.result();
          
            if (klass->major_version() < NOFAILOVER_MAJOR_VERSION &&
                FailOverToOldVerifier && !HAS_PENDING_EXCEPTION &&
                (exception_name == vmSymbols::java_lang_VerifyError() ||
                exception_name == vmSymbols::java_lang_ClassFormatError())) {
                
                exception_name = inference_verify(
                    klass, message_buffer, message_buffer_len, THREAD);
           }
        } else {
            exception_name = inference_verify(
            klass, message_buffer, message_buffer_len, THREAD);
        }
    }
    
    // 异常处理及返回， 省略
}
```

##### 符号引用验证
这个阶段的验证发生在解析阶段，用于检验对类自身以外（常量池中的各个符号引用）的信息进行匹配性校验，如：
* 符号引用中通过字符串描述的权限定名是否能找到对应的类
* 在指定类中是否存在符合方法描述符以及简单名称所描述的方法和字段
* ....

###### 目的
确保解析动作能正常执行。

###### 实现
在解析的实现中找。



#### 准备
* 为类静态变量分配内存并设置类静态变量初始值的阶段，这些变量都在方法区中进行分配。
* 对类静态变量初始化（使用“零值”）
* 不会执行任何字节码，包括类变量显式的初始化赋值语句
* 对于定义为`final`的类静态变量，javac会为这个字段生成`ConstantValue`属性，这此阶段会直接使用`ConstantValue`属性指定的值。

##### 实现
在Loaading的时候（`parseClassFile`方法）中：
```
// ....

// check that if this class is an interface then it doesn't have static methods
if (this_klass->is_interface()) {
  check_illegal_static_method(this_klass, CHECK_(nullHandle));
}

// Allocate mirror and initialize static fields
java_lang_Class::create_mirror(this_klass, CHECK_(nullHandle));
```
在解析字节流的时候，当部分验证完成后，调用方法`create_mirror`。它的执行过程主要关注下面2个方法：
```
void instanceKlass::do_local_static_fields_impl(instanceKlassHandle this_oop, void f(fieldDescriptor* fd, TRAPS), TRAPS) {
  for (JavaFieldStream fs(this_oop()); !fs.done(); fs.next()) {
    if (fs.access_flags().is_static()) {
      fieldDescriptor fd;
      fd.initialize(this_oop(), fs.index());
      f(&fd, CHECK);
    }
  }
}


static void initialize_static_field(fieldDescriptor* fd, TRAPS) {
  Handle mirror (THREAD, fd->field_holder()->java_mirror());
  assert(mirror.not_null() && fd->is_static(), "just checking");
  if (fd->has_initial_value()) {
    BasicType t = fd->field_type();
    switch (t) {
      case T_BYTE:
        mirror()->byte_field_put(fd->offset(), fd->int_initial_value());
              break;
      case T_BOOLEAN:
        mirror()->bool_field_put(fd->offset(), fd->int_initial_value());
              break;
      case T_CHAR:
        mirror()->char_field_put(fd->offset(), fd->int_initial_value());
              break;
      case T_SHORT:
        mirror()->short_field_put(fd->offset(), fd->int_initial_value());
              break;
      case T_INT:
        mirror()->int_field_put(fd->offset(), fd->int_initial_value());
        break;
      case T_FLOAT:
        mirror()->float_field_put(fd->offset(), fd->float_initial_value());
        break;
      case T_DOUBLE:
        mirror()->double_field_put(fd->offset(), fd->double_initial_value());
        break;
      case T_LONG:
        mirror()->long_field_put(fd->offset(), fd->long_initial_value());
        break;
      case T_OBJECT:
        {
          oop string = fd->string_initial_value(CHECK);
          mirror()->obj_field_put(fd->offset(), string);
        }
        break;
      default:
        THROW_MSG(vmSymbols::java_lang_ClassFormatError(),
                  "Illegal ConstantValue attribute in class file");
    }
  }
}
```
简单来说，就是通过`JavaFieldStream`遍历这个类的字段，遇到静态的，调用`initialize_static_field`方法，将初始值赋值到相应的Java对象。



#### 三. 解析
待补充。




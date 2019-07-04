---
layout: post
title: "HotSpot的虚函数表"
date: 2019-06-07 08:55
comments: false
tags: 
- JVM
categories:	
- JVM
---

本文研究**HotSpot的虚函数表**。

<!--more-->


### vtable的作用
<font color=orange>**实现Java的多态**</font>。通过虚函数表来实现运行期的方法分派。

### vtable放在哪里
以普通的Java类为例，在JVM里面有一个对应的 `instanceKlass`，而 **vtable** 就紧跟在这个 `instanceKlass`结构的后面。

代码如下：
```
/** instanceKlass 获取 vtable  */
klassVtable* instanceKlass::vtable() const {
  return new klassVtable(as_klassOop(), start_of_vtable(), vtable_length() / vtableEntry::size());
}


/**  instanceKlass 计算 vtable 的基址  */
intptr_t* start_of_vtable() const        { return ((intptr_t*)as_klassOop()) + vtable_start_offset(); }
static int vtable_start_offset()    { return header_size(); }
static int header_size()            { return align_object_offset(oopDesc::header_size() + sizeof(instanceKlass)/HeapWordSize); }


/** klassVtable 的部分代码   */
class klassVtable : public ResourceObj {
  KlassHandle  _klass;            // my klass
  int          _tableOffset;      // offset of start of vtable data within klass
  int          _length;           // length of vtable (number of entries)
  
public:
    klassVtable(KlassHandle h_klass, void* base, int length) : _klass(h_klass) {
        _tableOffset = (address)base - (address)h_klass(); _length = length;
    }
  
    // accessors
    vtableEntry* table() const      { return (vtableEntry*)(address(_klass()) + _tableOffset); }  
```

#### <font color=red>疑问：为什么需要加上oop的大小</font>?

计算vtable的起始偏移位置的代码（即通过 instanceKlass的大小 + oop的大小）。

为什么需要加上oop的大小呢？这里困扰了很久，因为和自己用HSDB看到的不一致。

<font color=green>问题的根源是：</font>
> HotSpot VM在JDK8之前的版本都是把Java对象和元数据对象以统一的方式由GC管理的。为了让GC能统一的处理这些对象，每个由GC管理的对象都继承自oopDesc，而每个oopDesc都有一个_klass字段指向描述它的Klass对象。GC在找到一个对象之后，要知道对象大小、对象里什么位置有GC需要知道的指针之类的信息，就会通过从_klass字段找到Klass对象，从Klass对象获取。更准确说Klass对象是嵌在klassOopDesc对象，以便Klass对象也同样得到GC的统一管理。

所以，我看的代码（版本是JDK1.7），instanceKlass是嵌在klassOopDesc对象中的，所以需要算上oop的大小。

列出一段日志：
```
InstanceKlass for java/lang/String @ 0x70e6c6a0 (object size = 384)  
 - _mark:    {0} :1  
 - _klass:   {4} :InstanceKlassKlass @ 0x70e60168  
 - _java_mirror:     {60} :Oop for java/lang/Class @ 0x70e77760  
 - _super:   {64} :InstanceKlass for java/lang/Object @ 0x70e65af8 
 ....
 
 
 InstanceKlassKlass @ 0x70e60168 (object size = 120)  
 - _mark:    {0} :1  
 - _klass:   {4} :KlassKlass @ 0x70e60000  
 - _java_mirror:     {60} :Oop for java/lang/Class @ 0x70e76f20  
 - _super:   {64} :null  
....


KlassKlass @ 0x70e60000 (object size = 120)  
 - _mark:    {0} :1  
 - _klass:   {4} :KlassKlass @ 0x70e60000  
 - _java_mirror:     {60} :Oop for java/lang/Class @ 0x70e76e00  
 - _super:   {64} :null  
 ....
```

回到这个小节的开始，vtable的位置如下：


<img src="/assets/blogImg/JVM/Vtable/1.png" width=300>


### vtable的长度是怎么定的

#### 步骤一
在解析class文件时（`ClassFileParser::parseClassFile`），通过调用` klassVtable::compute_vtable_size_and_num_mirandas`方法进行计算得到。大概逻辑是：
* 获取父类 vftable 的个数，并将当前类的 vftable 的个数设置为父类 vftable 的个数。
* 循环遍历当前 Java 类的每一个方法 ，判断是否需要为此方法在 vftable上添加1个entry。

#### 步骤二
当解析完class文件，会在内存创建一个 instanceKlass 对象用于存放类信息，具体看`instanceKlassKlass::allocate_instance_klass()`方法。
这个对象是在永久代(PerGen)分配内存的，分配内存的大小是 `oop.size + instanceKlass.size + vtable.size + itable.size + nonstatic_oop_map_size`。
这个vtable的长度也会写入`instanceKlass::_vtable_len`字段。



###  vtable的内容是如何写入的
在类的链接阶段（`instanceKlass::link_class_impl`），会初始化vtable(`klassVtable::initialize_vtable`)：
```
void klassVtable::initialize_vtable(bool checkconstraints, TRAPS) {
    // ... 省略
    
    int super_vtable_len = initialize_from_super(super);
    if (klass()->oop_is_array()) {
        assert(super_vtable_len == _length, "arrays shouldn't introduce new methods");
    } else {
        assert(_klass->oop_is_instance(), "must be instanceKlass");
    
        objArrayHandle methods(THREAD, ik()->methods());
        int len = methods()->length();
        int initialized = super_vtable_len;

        // update_inherited_vtable can stop for gc - ensure using handles
        for (int i = 0; i < len; i++) {
            HandleMark hm(THREAD);
            assert(methods()->obj_at(i)->is_method(), "must be a methodOop");
            methodHandle mh(THREAD, (methodOop)methods()->obj_at(i));

            bool needs_new_entry = update_inherited_vtable(ik(), mh, super_vtable_len, checkconstraints, CHECK);

            if (needs_new_entry) {
                put_method_at(mh(), initialized);
                mh()->set_vtable_index(initialized); // set primary vtable index
                initialized++;
            }
        }

        // add miranda methods; it will also update the value of initialized
        fill_in_mirandas(initialized);

        // In class hierarchies where the accessibility is not increasing (i.e., going from private ->
        // package_private -> publicprotected), the vtable might actually be smaller than our initial
        // calculation.
        assert(initialized <= _length, "vtable initialization failed");
        for(;initialized < _length; initialized++) {
            put_method_at(NULL, initialized);
        }
    }
}
```
上面的代码逻辑如下：
* 把父类的vtable拷贝到当前类
* 更新当前类的vtable，遍历当前类的所有方法：
  * 如果重写了父类的方法，那么更新vtable中对应的项
  * 如果没有重写父类的方法，那么在vtable新加一项
  

#### 拷贝父类的vtable
```
int klassVtable::initialize_from_super(KlassHandle super) {
    if (super.is_null()) {
        return 0;
    } else {
        // copy methods from superKlass
        // can't inherit from array class, so must be instanceKlass
        assert(super->oop_is_instance(), "must be instance klass");
        
        instanceKlass* sk = (instanceKlass*)super()->klass_part();
        klassVtable* superVtable = sk->vtable();
        assert(superVtable->length() <= _length, "vtable too short");
        
        // copy_vtable_to 相当于是 memory copy
        superVtable->copy_vtable_to(table());
        return superVtable->length();
    }
}
```

#### 更新当前类的vtable
```
// Update child's copy of super vtable for overrides
// OR return true if a new vtable entry is required
// Only called for instanceKlass's, i.e. not for arrays
// If that changed, could not use _klass as handle for klass
bool klassVtable::update_inherited_vtable(instanceKlass* klass, methodHandle target_method, int super_vtable_len,
                  bool checkconstraints, TRAPS) {
                  
    bool allocate_new = true;
    assert(klass->oop_is_instance(), "must be instanceKlass");
    
    // Initialize the method's vtable index to "nonvirtual".
    // If we allocate a vtable entry, we will update it to a non-negative number.
    /** 先给它赋一个负值， 等到给这个方法分配一个vtable的entry时，会更新index为正值 */
    target_method()->set_vtable_index(methodOopDesc::nonvirtual_vtable_index);
    
    // Static and <init> methods are never in
    if (target_method()->is_static() || target_method()->name() ==  vmSymbols::object_initializer_name()) {
        return false;
    }

    if (klass->is_final() || target_method()->is_final()) {
        // a final method never needs a new entry; final methods can be statically
        // resolved and they have to be present in the vtable only if they override
        // a super's method, in which case they re-use its entry
        allocate_new = false;
    }

    // we need a new entry if there is no superclass
    if (klass->super() == NULL) {
        return allocate_new;
    }
  
    // private methods always have a new entry in the vtable
    // specification interpretation since classic has
    // private methods not overriding
    if (target_method()->is_private()) {
        return allocate_new;
    }
    
    
    // search through the vtable and update overridden entries
    // Since check_signature_loaders acquires SystemDictionary_lock
    // which can block for gc, once we are in this loop, use handles
    // For classfiles built with >= jdk7, we now look for transitive overrides
    Symbol* name = target_method()->name();
    Symbol* signature = target_method()->signature();
    Handle target_loader(THREAD, _klass->class_loader());
    Symbol*  target_classname = _klass->name();
  
    for(int i = 0; i < super_vtable_len; i++) {
        methodOop super_method = method_at(i);
        
        // Check if method name matches
        if (super_method->name() == name && super_method->signature() == signature) {

            // get super_klass for method_holder for the found method
            instanceKlass* super_klass =  instanceKlass::cast(super_method->method_holder());

            /** 判断是否重写了父类（父父类）的方法 */
            if (
                (super_klass->is_override(super_method, target_loader, target_classname, THREAD)) 
                ||
                ((klass->major_version() >= VTABLE_TRANSITIVE_OVERRIDE_VERSION)  && 
                ((super_klass = find_transitive_override(super_klass, target_method, i, target_loader, target_classname, THREAD)) != (instanceKlass*)NULL))
             ) {
            
                // overriding, so no new entry
                allocate_new = false;
    
                // ... 省略
             
                // 将方法对应的 methodOop 设置在 vtable 的第 i 项
                put_method_at(target_method(), i);
                target_method()->set_vtable_index(i);
            }
        }
    }
    return allocate_new;
}                  
```



### 例子分析

跑以下代码，断点到最后一行：
```
public class Animal {
  public void say(){
    System.out.println("Animal :: say");
  }
  public void eat(){
    System.out.println("Animal :: eat");
  }
}

public class Cat extends Animal {
  public void say() {
    System.out.println("Cat :: say");
  }
  public void eat() {
    System.out.println("Cat :: eat");
  }
}

public class Dog extends Animal{
  public void say() {
    System.out.println("Dog :: say");
  }
  public void eat() {
    System.out.println("Dog :: eat");
  }
}

public class Test {

  public static void main(String[] args){
    Animal animal = new Animal();
    animal.say();
    animal.eat();

    Animal cat = new Cat();
    cat.say();
    cat.eat();

    Animal dog = new Dog();
    dog.say();
    dog.eat();

    System.out.println("");
  }
}
```

使用 HSDB 分别得到几个类的 InstanceKlass 的信息：
```
Object @ 0x00000007c0000f28
public void <init>() @0x000000010561a480;
static void <clinit>() @0x000000010561aca8;
protected void finalize() @0x000000010561ac10;
public final void wait(long, int) @0x000000010561aae0;
public final native void wait(long) @0x000000010561aa00;
public final void wait() @0x000000010561ab78;
public boolean equals(java.lang.Object) @0x000000010561a6e8;
public java.lang.String toString() @0x000000010561a840;
public native int hashCode() @0x000000010561a640;
public final native java.lang.Class getClass() [signature ()Ljava.lang.Class<*>;] @0x000000010561a5a8;
protected native java.lang.Object clone() @0x000000010561a778;
private static native void registerNatives() @0x000000010561a508;
public final native void notify() @0x000000010561a8c8;
public final native void notifyAll() @0x000000010561a960;

Animal @ 0x00000007c0062470
public void <init>() @0x0000000105a2a210;
public void say() @0x0000000105a2a2b8;
public void eat() @0x0000000105a2a360;
_vtable_len = 7

Cat @ 0x00000007c0062670
public void <init>() @0x0000000105a2a5e0;
public void say() @0x0000000105a2a688;
public void eat() @0x0000000105a2a730;
_vtable_len = 7

Dog @ 0x00000007c0062870
public void <init>() @0x0000000105a2a9b0;
public void say() @0x0000000105a2aa58;
public void eat() @0x0000000105a2ab00;
_vtable_len = 7
```

对于 Animal 的地址，查看它的vtable：

vtable的地址是 Animal的InstanceKlass地址 + sizeof(InstanceKlass) ,于是得到 0x7c0062470 + 0x1B8 = 0x7C0062628。

通过 在console 输入命令 `mem 0x7C0062628 7` ，得到：
```
0x00000007c0062628: 0x000000010561ac10  --> finalize
0x00000007c0062630: 0x000000010561a6e8  --> equals
0x00000007c0062638: 0x000000010561a840  --> toString
0x00000007c0062640: 0x000000010561a640  --> hashCode
0x00000007c0062648: 0x000000010561a778  --> clone
0x00000007c0062650: 0x0000000105a2a2b8  --> say
0x00000007c0062658: 0x0000000105a2a360  --> eat
```

同样方法查看 Cat 的 vtable : 
```
0x00000007c0062828: 0x000000010561ac10 --> finalize
0x00000007c0062830: 0x000000010561a6e8 --> equals
0x00000007c0062838: 0x000000010561a840 --> toString
0x00000007c0062840: 0x000000010561a640 --> hashCode
0x00000007c0062848: 0x000000010561a778 --> clone
0x00000007c0062850: 0x0000000105a2a688 --> say
0x00000007c0062858: 0x0000000105a2a730 --> eat
```

查看 Dog 的 vtable :
```
0x00000007c0062828: 0x000000010561ac10 --> finalize
0x00000007c0062830: 0x000000010561a6e8 --> equals
0x00000007c0062838: 0x000000010561a840 --> toString
0x00000007c0062840: 0x000000010561a640 --> hashCode
0x00000007c0062848: 0x000000010561a778 --> clone
0x00000007c0062a50: 0x0000000105a2aa58 --> say
0x00000007c0062a58: 0x0000000105a2ab00 --> eat
```

### 总结
1. vtable 分配在 instanceKlass对象实例的内存末尾 。
2. vtable可以看作是一个数组，数组中的每一项成员元素都是一个指针，指针指向 Java 方法在 JVM 内部所对应的 method 实例对象的内存首地址 。
3. vtable是 Java 实现面向对象的多态性的机制，如果一个 Java 方法可以被继承和重写， 则最终通过 put_method_at函数将方法地址替换,完成 Java 方法的动态绑定。
4. Java 子类会继承父类的 vtable，Java 中所有类都继承自 java.lang.Object。如果一个 Java 类中不声明任何方法，则其 vtalbe 的长度默认为 5。


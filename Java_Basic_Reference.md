---
layout: post
title: "JAVA的引用"
date: 2018-5-02 21:36
comments: false
tags: 
- Java
categories:	
- 基础
- Java
---

JAVA中对象的引用有四种，它们由高到低依次为: 强引用、软引用、弱引用和虚引用。
<!--more-->

### 几种引用的介绍

#### Strong Reference(强引用)
 * 强引用(StrongReference)强引用是使用最普遍的引用。
 * 如果一个对象具有强引用,那垃圾回收器绝不会回收它。
 * 当内存空间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足的问题。


#### SoftReference(软引用)
 * 如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它;
 * 如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。
 * 软引用可用来实现内存敏感的高速缓存。
 * 软引用可以和一个引用队列（ReferenceQueue）联合使用,如果软引用所引用的对象被垃圾回收器回收,Java虚拟机就会把这个软引用加入到与之关联的引用队列中。


#### WeakReference(弱引用)
 * 只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。
 * 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。


#### PhantomReference(虚引用)
 * 顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。
 * 虚引用主要用来跟踪对象被垃圾回收器回收的活动。
 * 虚引用与软引用和弱引用的一个区别在于: 虚引用必须和引用队列（ReferenceQueue）联合使用
 * 当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中。


###  WeakReference的用法

```bash
//创建一个强引用 
String str = new String("hello");
//创建引用队列, 为范型标记，表明队列中存放String对象的引用
ReferenceQueue rq = new ReferenceQueue();
//创建一个弱引用，它引用"hello"对象，并且与rq引用队列关联
WeakReference wf = new WeakReference(str, rq);
```

当前对象的关系:
1. 在堆区存在 : hello 对象 , WeakReference对象 , ReferenceQueue对象。
2.  hello 对象被 str 强引用(str在栈), 同时被 WeakReference对象弱引用着。因此，hello 对象不会被GC回收。


通过`str=null`取消 hello 对象的强引用，对象的关系(假设GC操作还没有进行)：
1. 在堆区存在 :  hello 对象 , WeakReference对象 , ReferenceQueue对象。
2.  hello 对象只被 WeakReference对象弱引用着。因此，hello 对象可能被GC回收。


假设 hello 对象没有被GC:
```bash
// 我们可以再次强引用 "hello"对象
String str1 = (String)wf.get();
// 从ReferenceQueue中poll()会得到Null
Reference ref = rq.poll();
```

假设 hello 对象已经被GC:
```bash
// 已经不能通过 WeakReference 得到 hello 对象 , o == null
Object o = wf.get();
// 从ReferenceQueue中poll()会得到 WeakReference对象
Reference ref = rq.poll();
```

###  使用软引用构造简单缓存
```bash
class Employee{
    private String id;// 雇员的标识号码    
    private String name;// 雇员姓名    
    private String department;// 该雇员所在部门    
    private String Phone;// 该雇员联系电话    
    private int salary;// 该雇员薪资    
    private String origin;// 该雇员信息的来源    
      
    // 构造方法     
    public Employee(String id) {     
        this.id = id;     
        getDataFromlnfoCenter();     
    }
    
    public String getId() {
        return id;
    }
    public void setId(String id) {
        this.id = id;
    }

    // 到数据库中取得雇员信息     
    private void getDataFromlnfoCenter() {     
        // 和数据库建立连接井查询该雇员的信息，将查询结果赋值     
        // 给name，department，plone，salary等变量     
        // 同时将origin赋值为"From DataBase"  
    }
}

/**
 *  目的:
 *    一个雇员信息查询系统的实例.
 * 
 *  分析:
 *    通过把过去查看过的雇员信息保存在内存中来提高性能。
 *    只有当内存吃紧时,才GC掉缓存的信息。
 */
class EmployeeCache {
	
    private HashMap employeeRefMap;
    private ReferenceQueue queue;
	
    private class EmployeeRef extends SoftReference{
        private String key = "";
		
        public EmployeeRef(Employee e, ReferenceQueue q) {
            super(e,q);
            key = e.getId();
        }
    }
	
    // 构造函数
    public EmployeeCache() {
        this.queue = new ReferenceQueue();
        this.employeeRefMap = new HashMap();
    }
	
    // 创建软引用指向 Employee对象,并保存此软引用.
    public void cacheEmployee(Employee e){
        cleanCache();
        EmployeeRef sf = new EmployeeRef(e,queue);
        employeeRefMap.put(e.getId(), sf);
	}
	
    //根据给定的ID,返回 Employee对象。
    public Employee getEmployee(String id){
        Employee e = null;
		
        // 缓存中是否有该Employee实例的软引用，如果有，从软引用中取得。
        if(employeeRefMap.containsKey(id)){
            System.out.println("get from cache , ID= "+id);
            EmployeeRef employeeref = (EmployeeRef)employeeRefMap.get(id);
            e = (Employee)employeeref.get();
        }

        // 如果软引用不存在或者软引用所指向的对象不存在,则重新创建一个对象
        if(e==null){
            e = new Employee(id);
            System.out.println("Retrieve From EmployeeInfoCenter. ID=" + e.getId());
            // 保存到缓存
            cacheEmployee(e);
        }
        return e;
    }
	
    // 清除那些软引用指向的Employee对象已经被回收的SoftReference对象
    private void cleanCache() {
        EmployeeRef sf = null;
        while((sf = (EmployeeRef)queue.poll())!=null){
            System.out.println(sf.key+"已经被回收");
            employeeRefMap.remove(sf.key);
        }
    }
}


/*以 VM参数 -Xmx2m 运行*/
public class SoftReferenceExample {

    public static void main(String[] args) {

        PrintStream out = new PrintStream("test.txt");
        System.setOut(out);

        Employee e = null;
        EmployeeCache eCache = new EmployeeCache();
		
        for(int i=0;i<=100000;i++){
            e = new Employee(String.valueOf(i));
            eCache.cacheEmployee(e);
        }
		
        for(int i=100000;i>=0;i--){
            e = eCache.getEmployee(String.valueOf(i));
            e = null;
        }
    }
}
```
【小小提示】
在cacheEmployee()方法里面的cleanCache()方法的调用是不能省略的，一旦省略了，就会导致过多的EmployeeRef对象残留在堆内存，一样会造成outOfMemoryError。
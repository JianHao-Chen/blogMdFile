---
layout: post
title: "JDK的动态代理"
date: 2018-05-12 10:55
comments: false
tags: 
- IO
- Java
categories:	
- 基础
- Java
---


动态代理，就是在运行期中，遵循Java编译系统组织.class文件的格式和结构，生成相应的二进制数据，然后再把这个二进制数据加载转换成对应的类。

<!--more-->

## JDK的动态代理

### ★ java.lang.reflect.Proxy
这是 Java 动态代理机制的主类，它提供了一组静态方法来为一组接口动态地生成代理类及其对象。

Proxy的几个重要方法:

``` 
/* 方法1 : 该方法用于获取指定代理对象所关联的调用处理器*/
public static InvocationHandler getInvocationHandler(Object proxy)

/* 方法2 : 该方法用于获取关联于指定类装载器和一组接口的动态代理类的类对象*/
static Class getProxyClass(ClassLoader loader, Class[] interfaces)

/* 方法3 : 该方法用于判断指定类对象是否是一个动态代理类*/
static boolean isProxyClass(Class cl)

/* 方法4 : 该方法用于为指定类装载器、一组接口及调用处理器生成动态代理类实例*/
static Object newProxyInstance(ClassLoader loader, Class[] interfaces,InvocationHandler h)
参数 :
  loader : 
    一个ClassLoader对象，定义了由哪个ClassLoader对象来对生成的代理对象进行加载

  interfaces :
    一个Interface对象的数组，表示的是我将要给我需要代理的对象提供一组什么接口，如果我提供了一组接口给它，那么这个代理对象就宣称实现了该接口(多态)，这样我就能调用这组接口中的方法了

  h :
    一个InvocationHandler对象，表示的是当我这个动态代理对象在调用方法的时候，会关联到哪一个InvocationHandler对象上
```


### ★ java.lang.reflect.InvocationHandler
这是调用处理器接口,它自定义了一个 invoke 方法,用于集中处理在动态代理类对象上的方法调用,通常在该方法中实现对委托类的代理访问。

InvocationHandler的核心方法:

```
Object invoke(Object proxy, Method method, Object[] args) throws Throwable
参数 :
  proxy:  
    指动态生成的代理对象

  method: 
    指代的是我们所要调用真实对象的某个方法的Method对象

  args: 
    指代的是调用真实对象某个方法时接受的参数
```

## Proxy的部分源码分析

###  ※※ Proxy 的重要静态变量 ※※ 
```
/* 映射表 : 用于维护类装载器对象到其对应的代理类缓存*/
private static Map loaderToCache = new WeakHashMap();

/* 标记  :  用于标记一个动态代理类正在被创建中*/
private static Object pendingGenerationMarker = new Object();

/* 同步表 : 记录已经被创建的动态代理类类型，主要被方法 isProxyClass 进行相关的判断*/
private static Map proxyClasses = Collections.synchronizedMap(new WeakHashMap()); 

/* 关联的调用处理器引用*/
protected InvocationHandler h;
```

###  ※※ Proxy 的构造方法 ※※
```
// 由于 Proxy 内部从不直接调用构造函数，所以 private 类型意味着禁止任何调用
private Proxy() {}

// 由于 Proxy 内部从不直接调用构造函数，所以 protected 意味着只有子类可以调用
protected Proxy(InvocationHandler h) {this.h = h;}
```


###  ※※ Proxy 的newProxyInstance方法 ※※
```
public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,
  InvocationHandler h){

  // 检查 h 不为空，否则抛异常
  if (h == null) throw new NullPointerException();

  // 获得与制定类装载器和一组接口相关的代理类类型对象
  Class cl = getProxyClass(loader, interfaces);

  // 通过反射获取构造函数对象并生成代理类实例
  try {
    Constructor cons = cl.getConstructor(constructorParams);
    return (Object) cons.newInstance(new Object[] { h });
  }
  catch(Exception e){}
}
```
动态代理真正的关键是在 getProxyClass 方法。

###  <font color=orange>※※ Proxy 的getProxyClass方法 ※※</font>
该方法负责为一组接口动态地生成代理类类型对象。
该方法总共可以分为四个步骤:

#### 步骤1
对这组接口进行一定程度的检查,包括:

 * 检查接口的数目是否<= 65535。
 * 检查接口类对象是否对类装载器可见并且与类装载器所能识别的接口类对象是完全相同的。
 * 还会检查确保是 interface 类型而不是 class 类型。
 * 确保所实现的接口没有重复。

```
if (interfaces.length > 65535) {
  throw new IllegalArgumentException("interface limit exceeded");

Class proxyClass = null;
/* collect interface names to use as key for proxy class cache */
String[] interfaceNames = new String[interfaces.length];

Set interfaceSet = new HashSet();	// for detecting duplicates
for (int i = 0; i < interfaces.length; i++) {
  /*
  * Verify that the class loader resolves the name of this
  * interface to the same Class object.
  */
  String interfaceName = interfaces[i].getName();
  Class interfaceClass = null;
  try {
    // 指定接口名字、类装载器对象，同时制定 initializeBoolean 为 false 表示无须初始化类
    // 如果方法返回正常这表示可见，否则会抛出 ClassNotFoundException 异常表示不可见
    interfaceClass = Class.forName(interfaceName, false, loader);
  } catch (ClassNotFoundException e) {}

  if (interfaceClass != interfaces[i]) {
    throw new IllegalArgumentException(
      interfaces[i] + " is not visible from class loader");
  }

  /*
   * Verify that the Class object actually represents an
   * interface.
   */
   if (!interfaceClass.isInterface()) {
     throw new IllegalArgumentException(
       interfaceClass.getName() + " is not an interface");
}

  /*
   * Verify that this interface is not a duplicate.
   */
  if (interfaceSet.contains(interfaceClass)) {
    throw new IllegalArgumentException(
      "repeated interface: " + interfaceClass.getName());
  }
  interfaceSet.add(interfaceClass);

  interfaceNames[i] = interfaceName;
}
```

#### 步骤2

1. 获取缓存表
从 loaderToCache 映射表中获取以类装载器对象为关键字所对应的缓存表，如果不存在就创建一个新的缓存表并更新到 loaderToCache。
```bash
Object key = Arrays.asList(interfaceNames);

/*
 * Find or create the proxy class cache for the class loader.
 */
Map cache;
synchronized (loaderToCache) {
  cache = (Map) loaderToCache.get(loader);
  if (cache == null) {
    cache = new HashMap();
    loaderToCache.put(loader, cache);
  }

  /*
   * This mapping will remain valid for the duration of this
   * method, without further synchronization, because the mapping
   * will only be removed if the class loader becomes unreachable.
   */
}
```
2. 从缓存表中获取接口对应的代理类的类对象引用
缓存表是一个 HashMap 实例，正常情况下它将存放键值对（接口名字列表，动态生成的代理类的类对象引用）。当代理类正在被创建时它会临时保存（接口名字列表，pendingGenerationMarker）。标记 pendingGenerationMarke 的作用是通知后续的同类请求（接口数组相同且组内接口排列顺序也相同）代理类正在被创建，请保持等待直至创建完成。

```
synchronized (cache) {
    /*
     * Look up the list of interfaces in the proxy class cache using
     * the key.  This lookup will result in one of three possible
     * kinds of values:
     *     1.null:
     *            if there is currently no proxy class for the list of
     *            interfaces in the class loader,
     *     2. the pendingGenerationMarker object:
     *            if a proxy class for the list of interfaces is currently
     *            being generated,
     *     3. a weak reference to a Class object:
     *            if a proxy class for the list of interfaces has already
     *            been generated.
     */

  do {
      Object value = cache.get(key);
      if (value instanceof Reference) {
        proxyClass = (Class) ((Reference) value).get();
    }
      if (proxyClass != null) {
        // proxy class already generated: return it
        return proxyClass;
    }else if (value == pendingGenerationMarker) {
        // proxy class being generated: wait for it
        try {
          cache.wait();
        }
        catch (InterruptedException e) {
          /*
           * The class generation that we are waiting for should
           * take a small, bounded time, so we can safely ignore
           * thread interrupts here.
           */
        }
        continue;
    }
    else {
        /*
         * No proxy class for this list of interfaces has been
         * generated or is being generated, so we will go and
         * generate it now.  Mark it as pending generation.
         */
        cache.put(key, pendingGenerationMarker);
        break;
    }
  }
  while (true);
}
```

#### 步骤3
动态创建代理类的类对象。

```
// 动态地生成代理类的字节码数组
byte[] proxyClassFile = ProxyGenerator.generateProxyClass( proxyName, interfaces); 
try {
  // 动态地定义新生成的代理类
  proxyClass = defineClass0(loader, proxyName, proxyClassFile, 0,proxyClassFile.length);
}catch (ClassFormatError e) {
  throw new IllegalArgumentException(e.toString());
}

// 把生成的代理类的类对象记录进 proxyClasses 表
proxyClasses.put(proxyClass, null);
```

#### 步骤4
根据结果更新缓存表，如果成功则将代理类的类对象引用更新进缓存表，否则清除缓存表中对应关键值，最后唤醒所有可能的正在等待的线程。
```
synchronized (cache) {
  if (proxyClass != null) {
    cache.put(key, new WeakReference(proxyClass));
  }
  else {
    cache.remove(key);
  }
  cache.notifyAll();
}
```


## ★ 动态代理的示例

JDK的动态代理创建机制是通过接口的。 比如现在想为RealSubject这个类创建一个动态代理对象，JDK主要会做以下工作：
1. 获取 RealSubject上的所有接口列表。
2. 确定要生成的代理类的类名，默认为：com.sun.proxy.$ProxyXXXX。
3. 根据需要实现的接口信息，在代码中动态创建 该Proxy类的字节码。
4. 将对应的字节码转换为对应的class 对象。
5. 创建InvocationHandler 实例handler，用来处理Proxy所有方法调用。
6. Proxy 的class对象 以创建的handler对象为参数，实例化一个proxy对象。


### 创建动态代理类

```
/*
 * 交通工具接口
 */
public interface Vehicle {
    public void drive();
}


/*
 * 可充电设备接口
 */
public interface Rechargable {
    public void recharge();
}


/*
 * 电能车类，实现Rechargable，Vehicle接口
 */
public class ElectricCar implements Rechargable, Vehicle{

    @Override
    public void drive() {
        System.out.println("Electric Car is Moving silently...");
    }

    @Override
    public void recharge() {
        System.out.println("Electric Car is Recharging...");
    }
}


public class InvocationHandlerImpl implements InvocationHandler{
    
    private ElectricCar car;
    
    public InvocationHandlerImpl(ElectricCar car){
        this.car=car;
    }
    
    public Object invoke(Object paramObject, Method paramMethod,
            Object[] paramArrayOfObject) throws Throwable {
        System.out.println("You are going to invoke "+paramMethod.getName()+" ...");  
        paramMethod.invoke(car, null);  
        System.out.println(paramMethod.getName()+" invocation Has Been finished...");  
        return null;
    }
}


public class Test {
    
    public static void main(String[] args) {
        ElectricCar car = new ElectricCar();
        
        // 1.获取对应的ClassLoader
        ClassLoader classLoader = car.getClass().getClassLoader();
        
        // 2.获取ElectricCar 所实现的所有接口
        Class[] interfaces = car.getClass().getInterfaces();
        
        // 3.设置一个来自代理传过来的方法调用请求处理器，处理所有的代理对象上的方法调用 
        InvocationHandler handler = new InvocationHandlerImpl(car);
        
      /*
       * 4.根据上面提供的信息，创建代理对象 在这个过程中:
       *   a.JDK会通过根据传入的参数信息动态地在内存中创建和 .class文件等同的字节码
       *   b.然后根据相应的字节码转换成对应的class
       *   c.然后调用newInstance()创建实例
       */
        Object o = Proxy.newProxyInstance(classLoader, interfaces, handler);
        Vehicle vehicle = (Vehicle) o;
        vehicle.drive();
        Rechargable rechargeable = (Rechargable) o;
        rechargeable.recharge();
    }
}
```

#### 生成动态代理类的字节码并且保存到硬盘中

```
public class ProxyUtils {

   /* 
    * 将根据类信息 动态生成的二进制字节码保存到硬盘中，默认的是clazz目录下 
    * params :clazz 需要生成动态代理类的类 
    * proxyName : 为动态生成的代理类的名称 
    */  
    public static void generateClassFile(Class clazz,String proxyName){
        //根据类信息和提供的代理类名称，生成字节码
        byte[] classFile = ProxyGenerator.generateProxyClass(
                proxyName, clazz.getInterfaces());
        String paths = clazz.getResource(".").getPath();
        System.out.println(paths);
        FileOutputStream out = null;
        
        try {
            //保存到硬盘中
            out = new FileOutputStream(paths+proxyName+".class");
            out.write(classFile);
            out.flush();
        }
        catch (Exception e) {
            e.printStackTrace();
        }
        finally { 
            try {
                out.close();
            }
            catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
    
    public static void main(String[] args){
        ElectricCar car = new ElectricCar();
        generateClassFile(car.getClass(), "ElectricCarProxy");
    }
}

```

#### 使用反编译工具jd-gui打开动态生成的代理类

```
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import jdkSrc.Proxy.example.Rechargable;
import jdkSrc.Proxy.example.Vehicle;

/*
 * 生成的动态代理类的组织模式是继承Proxy类，然后实现需要实现代理的类上的所有接口，
 * 而在实现的过程中，则是通过将所有的方法都交给了InvocationHandler来处理 
 */
public final class ElectricCarProxy extends Proxy
  implements Rechargable, Vehicle
{
  private static Method m1;
  private static Method m3;
  private static Method m4;
  private static Method m0;
  private static Method m2;

  public ElectricCarProxy(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }

  public final boolean equals(Object paramObject)
    throws 
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void recharge()
    throws 
  {
    try
    {
      this.h.invoke(this, m3, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void drive()
    throws 
  {
    try
    {
      this.h.invoke(this, m4, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final int hashCode()
    throws 
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String toString()
    throws 
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  static
  {
    try
    {
      // 为每一个需要方法对象，当调用相应的方法时，分别将方法对象作为参数传递给InvocationHandler处理
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("jdkSrc.Proxy.example.Rechargable").getMethod("recharge", new Class[0]);
      m4 = Class.forName("jdkSrc.Proxy.example.Vehicle").getMethod("drive", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```

可以看到，生成的动态代理类有以下特点:

1. 继承自 java.lang.reflect.Proxy，实现了 Rechargable,Vehicle 这两个ElectricCar实现的接口；
2. 类中的所有方法都是final 的；
3. 所有的方法功能的实现都统一调用了InvocationHandler的invoke()方法。


所以， <font color=red>**JDK的动态代理机制是基于接口的！**</font>
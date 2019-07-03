---
layout: post
title: "Reactor模型"
date: 2019-04-05 08:55
comments: false
tags: 
- IO
categories:	
- 基础
- CS
---

### Reactor 是什么
引用wiki上的介绍：
> The reactor design pattern is an event handling pattern for handling service requests delivered concurrently to a service handler by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to the associated request handlers.

可以得到如下关键点：
* 事件驱动（event handling）
* 可以处理一个或多个输入源（one or more inputs）
* 通过Service Handler同步的将输入事件（Event）采用多路复用分发给相应的Request Handler（多个）处理

<img height=400 src="/assets/blogImg/REACTOR/1.png">

<!--more-->

### Reactor 的实现
在应用Java NIO构建Reactor Pattern中，Doug Lea 在《Scalable IO in Java》中给了很好的阐述。里面介绍了3种Reactor。

#### 三种角色
* Reactor ：将I/O事件分派给对应的Handler
* Acceptor ：处理客户端新连接，并分派请求到处理器链中
* Handlers ：执行非阻塞读/写 任务

#### 单Reactor单线程模型
<img height=400 src="/assets/blogImg/REACTOR/2.png">


##### 说明
* Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发
* 如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后的后续业务处理
* 如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应
* Handler 会完成 Read→业务处理→Send 的完整业务流程

##### 代码示例
```
/**
 * 等待事件到来，分发事件处理
 */
class Reactor implements Runnable {

    private Reactor() throws Exception {
 
        SelectionKey sk =
                serverSocket.register(selector,
                        SelectionKey.OP_ACCEPT);
        // attach Acceptor 处理新连接
        sk.attach(new Acceptor());
    }

    public void run() {
        try {
            while (!Thread.interrupted()) {
                selector.select();
                Set selected = selector.selectedKeys();
                Iterator it = selected.iterator();
                while (it.hasNext()) {
                    it.remove();
                    //分发事件处理
                    dispatch((SelectionKey) (it.next()));
                }
            }
        } catch (IOException ex) {
            //do something
        }
    }

    void dispatch(SelectionKey k) {
        // 若是连接事件获取是acceptor
        // 若是IO读写事件获取是handler
        Runnable runnable = (Runnable) (k.attachment());
        if (runnable != null) {
            runnable.run();
        }
    }
}


/**
 * 连接事件就绪,处理连接事件
 */
class Acceptor implements Runnable {
    @Override
    public void run() {
        try {
            SocketChannel c = serverSocket.accept();
            if (c != null) {// 注册读写
                new Handler(c, selector);
            }
        } catch (Exception e) {}
    }
}


// 处理读写服务
class Handler implements Runnable {
    public void run() {
        try {
            //获取Socket的输入流，接收数据
            BufferedReader buf = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            String readData = buf.readLine();
            while (readData != null) {
                readData = buf.readLine();
                System.out.println(readData);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

##### <font color=red>注意</font>
这个模型下，所有事情全部都在一个线程中完成，很容易导致性能瓶颈。


#### 单 Reactor 多线程模型

<img height=400 src="/assets/blogImg/REACTOR/3.png">

##### 说明
* Reactor 对象通过 Select 监控客户端请求事件，收到事件后通过 Dispatch 进行分发
* 如果是建立连接请求事件，则由 Acceptor 通过 Accept 处理连接请求，然后创建一个 Handler 对象处理连接完成后续的各种事件
* 如果不是建立连接事件，则 Reactor 会分发调用连接对应的 Handler 来响应
* Handler 只负责响应事件，不做具体业务处理，通过 Read 读取数据后，会分发给后面的 Worker 线程池进行业务处理
* Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理
* Handler 收到响应结果后通过 Send 将响应结果返回给 Client

##### 代码示例
```
/**
 * 多线程处理读写业务逻辑
 */
class MultiThreadHandler implements Runnable {
    public static final int READING = 0, WRITING = 1;
    int state;
    final SocketChannel socket;
    final SelectionKey sk;

    //多线程处理业务逻辑
    ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());


    public MultiThreadHandler(SocketChannel socket, Selector sl) throws Exception {
        this.state = READING;
        this.socket = socket;
        sk = socket.register(selector, SelectionKey.OP_READ);
        sk.attach(this);
        socket.configureBlocking(false);
    }

    @Override
    public void run() {
        if (state == READING) {
            read();
        } else if (state == WRITING) {
            write();
        }
    }

    private void read() {
        //任务异步处理
        executorService.submit(() -> process());

        //下一步处理写事件
        sk.interestOps(SelectionKey.OP_WRITE);
        this.state = WRITING;
    }

    private void write() {
        //任务异步处理
        executorService.submit(() -> process());

        //下一步处理读事件
        sk.interestOps(SelectionKey.OP_READ);
        this.state = READING;
    }

    /**
      * task 业务处理
      */
    public void process() {
        //do IO ,task,queue something
    }
}
```

##### <font color=red>注意</font>
* 通过线程池处理业务逻辑，减小主reactor的性能开销
* Reactor承担所有事件的监听和响应，在单线程中运行，高并发场景下容易成为性能瓶颈。



#### 多Reactor多线程模型
<img height=400 src="/assets/blogImg/REACTOR/4.png">

##### 说明
* Reactor 主线程 MainReactor 对象通过 Select 监控建立连接事件，收到事件后交给 Acceptor
* Acceptor 处理建立连接事件后，MainReactor 将连接分配给 SubReactor 进行处理
* SubReactor 将连接加入连接队列进行监听，并创建一个 Handler 用于处理各种连接事件；
* 当有新的事件发生时，SubReactor 会调用连接对应的 Handler 进行响应
* Handler 通过 Read 读取数据后，会分发给后面的 Worker 线程池进行业务处理
* Worker 线程池会分配独立的线程完成真正的业务处理，如何将响应结果发给 Handler 进行处理
* Handler 收到响应结果后通过 Send 将响应结果返回给 Client

##### 代码示例
```
/**
 * 多work 连接事件Acceptor,处理连接事件
 */
class MultiWorkThreadAcceptor implements Runnable {

    // cpu线程数相同多work线程
    int workCount =Runtime.getRuntime().availableProcessors();
    SubReactor[] workThreadHandlers = new SubReactor[workCount];
    volatile int nextHandler = 0;

    public MultiWorkThreadAcceptor() {
        this.init();
    }

    public void init() {
        nextHandler = 0;
        for (int i = 0; i < workThreadHandlers.length; i++) {
            try {
                workThreadHandlers[i] = new SubReactor();
            } catch (Exception e) {
            }

        }
    }

    @Override
    public void run() {
        try {
            SocketChannel c = serverSocket.accept();
            if (c != null) {// 注册读写
                synchronized (c) {
                    // 顺序获取SubReactor，然后注册channel 
                    SubReactor work = workThreadHandlers[nextHandler];
                    work.registerChannel(c);
                    nextHandler++;
                    if (nextHandler >= workThreadHandlers.length) {
                        nextHandler = 0;
                    }
                }
            }
        } catch (Exception e) {
        }
    }
}


/**
 * 多work线程处理读写业务逻辑
 */
class SubReactor implements Runnable {
    final Selector mySelector;

    //多线程处理业务逻辑
    int workCount =Runtime.getRuntime().availableProcessors();
    ExecutorService executorService = Executors.newFixedThreadPool(workCount);


    public SubReactor() throws Exception {
        // 每个SubReactor 一个selector 
        this.mySelector = SelectorProvider.provider().openSelector();
    }

    /**
      * 注册chanel
      *
      * @param sc
      * @throws Exception
      */
    public void registerChannel(SocketChannel sc) throws Exception {
        sc.register(mySelector, SelectionKey.OP_READ | SelectionKey.OP_CONNECT);
    }

    @Override
    public void run() {
        while (true) {
            try {
            //每个SubReactor 自己做事件分派处理读写事件
                selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = keys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey key = iterator.next();
                    iterator.remove();
                    if (key.isReadable()) {
                        read();
                    } else if (key.isWritable()) {
                        write();
                    }
                }

            } catch (Exception e) {

            }
        }
    }

    private void read() {
        //任务异步处理
        executorService.submit(() -> process());
    }

    private void write() {
        //任务异步处理
        executorService.submit(() -> process());
    }

    /**
      * task 业务处理
      */
    public void process() {
        //do IO ,task,queue something
    }
}
```


#### 总结
Reactor 模式具有如下的优点：
* 响应快，不必为单个同步时间所阻塞，虽然 Reactor 本身依然是同步的
* 编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销
* 可扩展性，可以方便的通过增加 Reactor 实例个数来充分利用 CPU 资源
* 可复用性，Reactor 模型本身与具体事件处理逻辑无关，具有很高的复用性
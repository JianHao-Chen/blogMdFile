---
layout: post
title: "Linux的epoll"
date: 2018-03-03 19:12
comments: false
tags: 
- IO
- Linux
categories:	
- 基础
- CS

---

关于这篇文章：
* 内容：Linux的 epoll 的API和它的实现。
* 目的：学习Java Nio的过程中，想了解一下 epoll
* 注意：部分内容来自网上，不敢保证一定对。

<!--more-->


### epoll的介绍

#### epoll的3个接口

##### epoll_create
```
int epoll_create(int size)
```
创建一个 epoll 文件描述符,size指明这个epoll监听的数目有多大，实际上现在这个 size 是被忽略的（内核使用红黑树组织epoll相关数据结构，不再使用这个参数）。

##### epoll_ctl
```
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
入参解析：
* epfd ：1个文件描述符，引用的是 epoll 实例，也是这个系统调用操作的对象。
* fd ：操作的目标文件描述符
* op ： 表示操作的类型，可选的值有：
  * EPOLL_CTL_ADD ：将目标文件描述符 fd 添加到 epfd 引用的 epoll 实例，并且与 event关联起来。
  * EPOLL_CTL_MOD ：修改 fd 关联的 event 
  * EPOLL_CTL_DEL ：将目标文件描述符 fd 从 epfd 引用的 epoll 实例中删除，event参数被忽略。
  
* event ：是一个结构体，包含 event信息和用户自定义信息。
  ```
  // 这是一个union一次只能存储其中一种数据，可以是文件描述符fd，
  // 可以是传递的数据void*，可以是一个无符号长整形等等，但是最经常使用的是fd
  typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
  } epoll_data_t;

  struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
  };
  
  // events有以下几种：
  EPOLLIN ：表示对应的文件描述符可读(包括对端socket关闭)
  EPOLLOUT：表示对应的文件描述符可以写；
  EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）
  EPOLLERR：表示对应的文件描述符发生错误；
  EPOLLHUP：表示对应的文件描述符被挂断；
  EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的；
  EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里。
  ```
  
##### epoll_wait
```
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```
入参解析：
* epfd ： 引用的是 epoll 实例，也是这个系统调用操作的对象。
* events ：用于保存这个系统调用的返回值，即就绪的 event
* maxevents ：用户想监听的 fd 的数目
* timeout ： 超时时间 (0表示立即返回，-1表示永久阻塞，直到有就绪事件)


#### epoll 的 LT 和 ET

##### LT（Level Triggered）水平触发
LT是缺省的工作方式，并且同时支持block和no-block socket。在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表。

##### ET（Edge Triggered）边缘触发
ET是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知。请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)。

#### epoll的使用例子
```
#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];
int listen_sock, conn_sock, nfds, epollfd;

/* Set up listening socket, 'listen_sock' (socket(),
   bind(), listen()) */

epollfd = epoll_create(10);
if (epollfd == -1) {
    perror("epoll_create");
    exit(EXIT_FAILURE);
}

ev.events = EPOLLIN;
ev.data.fd = listen_sock;
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
    perror("epoll_ctl: listen_sock");
    exit(EXIT_FAILURE);
}

for (;;) {
    nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
        perror("epoll_pwait");
        exit(EXIT_FAILURE);
    }

   for (n = 0; n < nfds; ++n) {
        if (events[n].data.fd == listen_sock) {
            conn_sock = accept(listen_sock,
                            (struct sockaddr *) &local, &addrlen);
            if (conn_sock == -1) {
                perror("accept");
                exit(EXIT_FAILURE);
            }
            setnonblocking(conn_sock);
            ev.events = EPOLLIN | EPOLLET;
            ev.data.fd = conn_sock;
            if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                        &ev) == -1) {
                perror("epoll_ctl: conn_sock");
                exit(EXIT_FAILURE);
            }
        } else {
            do_use_fd(events[n].data.fd);
        }
    }
}
```


### epoll的实现

#### 相关数据结构
![](/assets/blogImg/linux_epoll/1.png)

介绍上面的数据结构。

##### epitem
当向系统中添加一个fd时，就创建一个epitem结构体，这是内核管理epoll的基本数据结构。
```
struct epitem {

    struct rb_node  rbn;        //用于主结构管理的红黑树
    struct list_head  rdllink;  //事件就绪队列
    struct epitem  *next;       //用于主结构体中的链表
    struct epoll_filefd  ffd;   //这个结构体对应的被监听的文件描述符信息
    int  nwait;                 //poll操作中事件的个数
    struct list_head  pwqlist;  //双向链表，保存着被监视文件的等待队列，功能类似于select/poll中的poll_table
    struct eventpoll  *ep;      //该项属于哪个主结构体（多个epitm从属于一个eventpoll）
    struct list_head  fllink;   //双向链表，用来链接被监视的文件描述符对应的struct file。因为file里有f_ep_link,用来保存所有监视这个文件的epoll节点
    struct epoll_event  event;  //注册的感兴趣的事件,也就是用户空间的epoll_event
}
```


##### eventpoll
每个epoll fd（epfd）对应的主要数据结构。
```
struct eventpoll {

    spin_lock_t       lock;        //对本数据结构的访问
    struct mutex      mtx;         //防止使用时被删除
    wait_queue_head_t     wq;      //sys_epoll_wait() 使用的等待队列
    wait_queue_head_t   poll_wait;       //file->poll()使用的等待队列
    struct list_head    rdllist;        //事件满足条件的链表
    struct rb_root      rbr;            //用于管理所有fd的红黑树（树根）
    struct epitem      *ovflist;       //将事件到达的fd进行链接起来发送至用户空间
}
```

#### eventpoll_init
epoll需要占用某些系统资源。Linux内核在系统启动时，其实已经提前完成了该部分资源的分配和初始化。

```
/* 本文的内核源码版本为linux-3.10.103 */

/* linux-3.10.103/linux-3.10.103/fs/eventpoll.c */

static int __init eventpoll_init(void)
{
    mutex_init(&pmutex);
    ep_poll_safewake_init(&psw);
    
    epi_cache = kmem_cache_create("eventpoll_epi", sizeof(struct epitem), 0, SLAB_HWCACHE_ALIGN|EPI_SLAB_DEBUG|SLAB_PANIC, NULL);

    pwq_cache = kmem_cache_create("eventpoll_pwq", sizeof(struct eppoll_entry), 0, EPI_SLAB_DEBUG|SLAB_PANIC, NULL);

    return 0;
}
```
“eventpoll_init()”在内核模块加载时将被调用来完成epoll相关的资源分配，从其源码可见，Linux内核为epoll提供了”eventpoll_epi”和”eventpoll_pwq”两块基于slab机制的内存单元,epoll中对象的分配和释放操作非常频繁，slab可以最大程度提升epoll使用内存的效率。


#### epoll_create()的实现
```
long sys_epoll_create(int size) {

    struct eventpoll *ep;
    ...
    ep_alloc(&ep); //为ep分配内存并进行初始化

    /* 
    * 调用anon_inode_getfd 新建一个file instance，也就是epoll可以看成一个文件（匿名文件）。
    * 因此我们可以看到epoll_create会返回一个fd。
    * epoll所管理的所有的fd都是放在一个大的结构eventpoll(红黑树)中，将主结构体 
    * struct eventpoll *ep 放入 file->private 项中进行保存（sys_epoll_ctl会取用）
    */

    fd = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep, O_RDWR | (flags & O_CLOEXEC));
     return fd;
}
```

#### epoll_ctl的实现
```
asmlinkage long sys_epoll_ctl(int epfd,int op,int fd,struct epoll_event __user *event) {

    int error;

    struct file *file,*tfile;

    struct eventpoll *ep;

    struct epoll_event epds;

    error = -FAULT;

    //判断参数的合法性，将 __user *event 复制给 epds。
    if(ep_op_has_event(op) && copy_from_user(&epds,event,sizeof(struct epoll_event)))
            goto error_return; //省略跳转到的代码

    file  = fget (epfd); // epoll fd 对应的文件对象

    tfile = fget(fd);    // fd 对应的文件对象

    //在create时存入进去的（anon_inode_getfd），现在取用。
    ep = file->private->data;

    mutex_lock(&ep->mtx);

    //防止重复添加（在ep的红黑树中查找是否已经存在这个fd）
    epi = epi_find(ep,tfile,fd);

    switch(op)
    {
       ...
        case EPOLL_CTL_ADD:  //增加监听一个fd
            if(!epi)
            {
                epds.events |= EPOLLERR | POLLHUP;     //默认包含POLLERR和POLLHUP事件

                error = ep_insert(ep,&epds,tfile,fd);  //在ep的红黑树中插入这个fd对应的epitm结构体。
            } 
            else  //重复添加（在ep的红黑树中查找已经存在这个fd）。
                error = -EEXIST;
            break;
        ...
    }
    return error;
}
```

ep_insert的实现：
```
static int ep_insert(struct eventpoll *ep, struct epoll_event *event, struct file *tfile, int fd)
{

   int error ,revents,pwake = 0;
   unsigned long flags ;
   struct epitem *epi;
   
   /*
      struct ep_queue{
         poll_table pt;
         struct epitem *epi;
      }
    */
   struct ep_pqueue epq;

   //分配一个epitem结构体来保存每个加入的fd
   if(!(epi = kmem_cache_alloc(epi_cache,GFP_KERNEL)))
      goto error_return;

   //初始化该结构体
   ep_rb_initnode(&epi->rbn);
   INIT_LIST_HEAD(&epi->rdllink);
   INIT_LIST_HEAD(&epi->fllink);
   INIT_LIST_HEAD(&epi->pwqlist);
   epi->ep = ep;
   ep_set_ffd(&epi->ffd,tfile,fd);
   epi->event = *event;
   epi->nwait = 0;
   epi->next = EP_UNACTIVE_PTR;

 
   epq.epi = epi;
   //安装poll回调函数
   init_poll_funcptr(&epq.pt, ep_ptable_queue_proc );

   /* 
   * 调用poll函数来获取当前事件位，其实是利用它来调用注册函数ep_ptable_queue_proc（poll_wait中调用）。
   * 如果fd是套接字，f_op为socket_file_ops，poll函数是 sock_poll()。
   * 如果是TCP套接字的话，进而会调用到tcp_poll()函数。
   * 此处调用poll函数查看当前文件描述符的状态，存储在revents中。
   * 在poll的处理函数(tcp_poll())中，会调用sock_poll_wait()，在sock_poll_wait()中会调用
     到epq.pt.qproc指向的函数，也就是ep_ptable_queue_proc()。  
    */ 
   revents = tfile->f_op->poll(tfile, &epq.pt);

   spin_lock(&tfile->f_ep_lock);
   list_add_tail(&epi->fllink,&tfile->f_ep_lilnks);
   spin_unlock(&tfile->f_ep_lock);

   ep_rbtree_insert(ep,epi); //将该epi插入到ep的红黑树中

   spin_lock_irqsave(&ep->lock,flags);

    //  revents & event->events：刚才fop->poll的返回值中标识的事件有用户event关心的事件发生。

    // !ep_is_linked(&epi->rdllink)：epi的ready队列中有数据。ep_is_linked用于判断队列是否为空。

    /*  
    * 如果要监视的文件状态已经就绪并且还没有加入到就绪队列中,则将当前的 
    * epitem加入到就绪队列中.如果有进程正在等待该文件的状态就绪,则唤醒一个等待的进程。
    */ 
    if((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
      list_add_tail(&epi->rdllink,&ep->rdllist); //将当前epi插入到ep->ready队列中。

    /* 
    * 如果有进程正在等待文件的状态就绪，也就是调用epoll_wait睡眠的进程正在等待，则唤醒一个等待进程。
    * waitqueue_active(q) 等待队列q中有等待的进程返回1，否则返回0。
    */
      if(waitqueue_active(&ep->wq))
         __wake_up_locked(&ep->wq,TAKS_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE);
         
        /*  
        * 如果有进程等待eventpoll文件本身（???）的事件就绪，则增加临时变量pwake的值，
        * pwake的值不为0时，在释放lock后，会唤醒等待进程。 
        */ 
      if(waitqueue_active(&ep->poll_wait))
         pwake++;
   }

   spin_unlock_irqrestore(&ep->lock,flags);

    if(pwake)
      ep_poll_safewake(&psw,&ep->poll_wait);    //唤醒等待eventpoll文件状态就绪的进程

   return 0;
}
```

上面代码中：
```
init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
revents = tfile->f_op->poll(tfile, &epq.pt);
```
这2个函数将ep_ptable_queue_proc注册到epq.pt中的qproc：
```
typedef struct poll_table_struct {
    poll_queue_proc qproc;
    unsigned long key;
}poll_table;
```

执行`f_op->poll(tfile, &epq.pt)`时，<font color=orange>XXX_poll(tfile, &epq.pt)</font> 函数会执行poll_wait()，poll_wait()会调用epq.pt.qproc函数，即ep_ptable_queue_proc。

ep_ptable_queue_proc函数如下：
```
/*
 * 在文件操作中的poll函数中调用，将epoll的回调函数加入到目标文件的唤醒队列中。
 * 如果监视的文件是套接字，参数whead则是sock结构的sk_sleep成员的地址。  
 */

static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead, poll_table *pt) {

    /* struct ep_queue{
        poll_table pt;
        struct epitem *epi;
      } */
    struct epitem *epi = ep_item_from_epqueue(pt); //pt获取struct ep_queue的epi字段。
    struct eppoll_entry *pwq;

    if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
        pwq->whead = whead;
        pwq->base = epi;
        add_wait_queue(whead, &pwq->wait);
        list_add_tail(&pwq->llink, &epi->pwqlist);
        epi->nwait++;
    } 
    else {

        /* We have to signal that an error occurred */
        /*
         * 如果分配内存失败，则将nwait置为-1，表示
         * 发生错误，即内存分配失败，或者已发生错误
         */
        epi->nwait = -1;
    }
}
```
`ep_ptable_queue_proc`函数完成`epitem`加入到特定文件的wait队列任务。

ep_ptable_queue_proc有三个参数：
* `struct file *file` ：  该fd对应的文件对象
* `wait_queue_head_t *whead` ： 该fd对应的设备等待队列（同select中的mydev->wait_address）
* `poll_table *pt;` ：f_op->poll(tfile, &epq.pt)中的epq.pt


在`ep_ptable_queue_proc`函数中，引入了另外一个非常重要的数据结构`eppoll_entry`。`eppoll_entry`<font color=orange>主要完成epitem和epitem事件发生时的callback（ep_poll_callback）函数之间的关联</font>。<font color=red>
首先将eppoll_entry的whead指向fd的设备等待队列（同select中的wait_address），然后初始化`eppoll_entry`的base变量指向epitem，最后通过`add_wait_queue`将`epoll_entry`挂载到fd的设备等待队列上。完成这个动作后，`epoll_entry`已经被挂载到fd的设备等待队列。</font>

其中struct eppoll_entry定义如下：
```
struct eppoll_entry {
    struct list_head llink;
    struct epitem *base;
    wait_queue_t wait;
    wait_queue_head_t *whead;
};
```

<font color=orange>由于`ep_ptable_queue_proc`函数设置了等待队列的`ep_poll_callback`回调函数。所以在设备硬件数据到来时，硬件中断处理函数中会唤醒该等待队列上等待的进程时，会调用唤醒函数`ep_poll_callback`</font>：
```
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key) {

    int pwake = 0;
    unsigned long flags;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;

    spin_lock_irqsave(&ep->lock, flags);

    //判断注册的感兴趣事件

    //#define EP_PRIVATE_BITS  (EPOLLONESHOT | EPOLLET)
    //有非EPOLLONESHONT或EPOLLET事件
    if (!(epi->event.events & ~EP_PRIVATE_BITS))
        goto out_unlock;

    if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
        if (epi->next == EP_UNACTIVE_PTR) {
            epi->next = ep->ovflist;
            ep->ovflist = epi;
      }
        goto out_unlock;
    }
 
    if (ep_is_linked(&epi->rdllink))
        goto is_linked;

    //***关键***，将该fd加入到epoll监听的就绪链表中
    list_add_tail(&epi->rdllink, &ep->rdllist);
    //唤醒调用epoll_wait()函数时睡眠的进程。用户层epoll_wait(...) 超时前返回。
    if (waitqueue_active(&ep->wq))
        __wake_up_locked(&ep->wq, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE);

    if (waitqueue_active(&ep->poll_wait))
        pwake++;

    out_unlock: spin_unlock_irqrestore(&ep->lock, flags);

    if (pwake)
        ep_poll_safewake(&psw, &ep->poll_wait);

    return 1;
}
```
所以`ep_poll_callback`函数主要的功能是<font color=orange>将被监视文件的等待事件就绪时，将文件对应的epitem实例添加到就绪队列中</font>，当用户调用`epoll_wait()`时，内核会将就绪队列中的事件报告给用户。


#### epoll_wait的实现
```
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events, int, maxevents, int, timeout)  {

    int error;
    struct file *file;
    struct eventpoll *ep;

    /* 检查maxevents参数。 */
    if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
        return -EINVAL;

    /* 检查用户空间传入的events指向的内存是否可写。参见__range_not_ok()。 */
    if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event))) {
        error = -EFAULT;
        goto error_return;
   }

    /* 获取epfd对应的eventpoll文件的file实例，file结构是在epoll_create中创建。 */
    error = -EBADF;
    file = fget(epfd);
    
    if (!file)
        goto error_return;

    /* 通过检查epfd对应的文件操作是不是eventpoll_fops 来判断epfd是否是一个eventpoll文件。如果不是则返回EINVAL错误。 */
    error = -EINVAL;

    if (!is_file_epoll(file))
        goto error_fput;

    /* At this point it is safe to assume that the "private_data" contains  */
    ep = file->private_data;

    /* Time to fish for events ... */
    error = ep_poll(ep, events, maxevents, timeout);

    error_fput:

    fput(file);

    error_return:

    return error;
}
```

`epoll_wait`调用`ep_poll`，`ep_poll`实现如下：
```
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout) {

    int res, eavail;
    unsigned long flags;
    long jtimeout;
    wait_queue_t wait;

    /* timeout是以毫秒为单位，这里是要转换为jiffies时间。这里加上999(即1000-1)，是为了向上取整。 */
    jtimeout = (timeout < 0 || timeout >= EP_MAX_MSTIMEO) ?MAX_SCHEDULE_TIMEOUT : (timeout * HZ + 999) / 1000;

retry:
    spin_lock_irqsave(&ep->lock, flags);
    res = 0;
    
    if (list_empty(&ep->rdllist)) {
    /* 没有事件，所以需要睡眠。当有事件到来时，睡眠会被ep_poll_callback函数唤醒。*/

    init_waitqueue_entry(&wait, current); //将current进程放在wait这个等待队列中。
    wait.flags |= WQ_FLAG_EXCLUSIVE;

    /* 将当前进程加入到eventpoll的等待队列中，等待文件状态就绪或直到超时，或被信号中断。 */
    __add_wait_queue(&ep->wq, &wait);

    for (;;) {

        /* 执行ep_poll_callback()唤醒时应当需要将当前进程唤醒，所以当前进程状态应该为“可唤醒”TASK_INTERRUPTIBLE  */
        set_current_state(TASK_INTERRUPTIBLE);

        /* 如果就绪队列不为空，也就是说已经有文件的状态就绪或者超时，则退出循环。*/
        if (!list_empty(&ep->rdllist) || !jtimeout)
            break;

        /* 如果当前进程接收到信号，则退出循环，返回EINTR错误 */
        if (signal_pending(current)) {
            res = -EINTR;
            break;
        }

        spin_unlock_irqrestore(&ep->lock, flags);

        /* 
        * 主动让出处理器，等待ep_poll_callback()将当前进程唤醒或者超时,返回值是剩余的时间。
        * 从这里开始当前进程会进入睡眠状态，直到某些文件的状态就绪或者超时。
        * 当文件状态就绪时，eventpoll的回调函数ep_poll_callback()会唤醒在ep->wq指向的等待队列中的进程。
        */
        jtimeout = schedule_timeout(jtimeout);
        spin_lock_irqsave(&ep->lock, flags);
      }

        __remove_wait_queue(&ep->wq, &wait);
        set_current_state(TASK_RUNNING);
    }

    /* 
    * ep->ovflist链表存储的向用户传递事件时暂存就绪的文件。
    * 所以不管是就绪队列ep->rdllist不为空，或者ep->ovflist不等于
    * EP_UNACTIVE_PTR，都有可能现在已经有文件的状态就绪。
    * ep->ovflist不等于EP_UNACTIVE_PTR有两种情况，一种是NULL，此时
    * 可能正在向用户传递事件，不一定就有文件状态就绪，
    * 一种情况时不为NULL，此时可以肯定有文件状态就绪，
    * 参见ep_send_events()。
    */
    eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
    
    spin_unlock_irqrestore(&ep->lock, flags);

    /* Try to transfer events to user space. In case we get 0 events and there's still timeout left over, we go trying again in search of more luck. */
    /* 
    * 如果没有被信号中断，并且有事件就绪，但是没有获取到事件(有可能被其他进程获取到了)，并且没有超时，
    * 则跳转到retry标签处，重新等待文件状态就绪。 
    */
    if (!res && eavail && !(res = ep_send_events(ep, events, maxevents)) && jtimeout)
        goto retry;

    /* 返回获取到的事件的个数或者错误码 */
    return res;
}
```


#### 总结

##### epoll为什么高效（相比select）：
* 传递 fd ： select/poll每次调用都要传递所要监控的所有fd给select/poll系统调用（这意味着每次调用都要将fd列表从用户态拷贝到内核态，当fd数目很多时，这会造成低效）。而每次调用epoll_wait时（作用相当于调用select/poll），不需要再传递fd列表给内核，因为已经在epoll_ctl中将需要监控的fd告诉了内核（epoll_ctl不需要每次都拷贝所有的fd，只需要进行增量式操作）。所以，在调用epoll_create之后，内核已经在内核态开始准备数据结构存放要监控的fd了。每次epoll_ctl只是对这个数据结构进行简单的维护。

* epoll使用了slab机制 ：在内核里，一切皆文件。所以，epoll向内核注册了一个文件系统，用于存储上述的被监控的fd。当你调用epoll_create时，就会在这个虚拟的epoll文件系统里创建一个file结点。当然这个file不是普通文件，它只服务于epoll。epoll在被内核初始化时（操作系统启动），同时会开辟出epoll自己的内核高速cache区，用于安置每一个我们想监控的fd，这些fd会以红黑树的形式保存在内核cache里，以支持快速的查找、插入、删除。这个内核高速cache区，就是建立连续的物理内存页，然后在之上建立slab层，简单的说，就是物理上分配好你想要的size的内存对象，每次使用时都是使用空闲的已分配好的对象。 
* 遍历 fd : 当我们调用epoll_ctl往里塞入百万个fd时，epoll_wait仍然可以飞快的返回，并有效的将发生事件的fd给我们用户。这是由于我们在调用epoll_create时，内核除了帮我们在epoll文件系统里建了个file结点，在内核cache里建了个红黑树用于存储以后epoll_ctl传来的fd外，还会再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据即可。有数据就返回，没有数据就sleep，等到timeout时间到后即使链表没数据也返回。所以，epoll_wait非常高效。而且，通常情况下即使我们要监控百万计的fd，大多一次也只返回很少量的准备就绪fd而已，所以，epoll_wait仅需要从内核态copy少量的fd到用户态而已。那么，这个准备就绪list链表是怎么维护的呢？当我们执行epoll_ctl时，除了把fd放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个fd的中断到了，就把它放到准备就绪list链表里。所以，当一个fd（例如socket）上有数据到了，内核在把设备（例如网卡）上的数据copy到内核中后就来把fd（socket）插入到准备就绪list链表里了。

##### 3个系统调用的概括
1. 执行epoll_create时，创建了红黑树和就绪list链表。
2. 执行epoll_ctl时，如果增加fd（socket），则检查在红黑树中是否存在，存在立即返回，不存在则添加到红黑树上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪list链表中插入数据。
3. 执行epoll_wait时立刻返回准备就绪链表里的数据即可。


#### 补充
epoll有两种模式LT(水平触发)和ET(边缘触发)，LT模式下，主要缓冲区数据一次没有处理完，那么下次epoll_wait返回时，还会返回这个句柄；而ET模式下，缓冲区数据一次没处理结束，那么下次是不会再通知了，只在第一次返回．所以在ET模式下，一般是通过while循环，一次性读完全部数据．epoll默认使用的是LT。

这件事怎么做到的呢？当一个socket句柄上有事件时，内核会把该句柄插入上面所说的准备就绪list链表，这时我们调用epoll_wait，会把准备就绪的socket拷贝到用户态内存，然后清空准备就绪list链表，最后，epoll_wait干了件事，就是检查这些socket，如果不是ET模式（就是LT模式的句柄了），并且这些socket上确实有未处理的事件时，又把该句柄放回到刚刚清空的准备就绪链表了。所以，非ET的句柄，只要它上面还有事件，epoll_wait每次都会返回。而ET模式的句柄，除非有新中断到，即使socket上的事件没有处理完，也是不会次次从epoll_wait返回的．

经常看到比较ET和LT模式到底哪个效率高的问题．有一个回答是说ET模式下减少epoll系统调用．这话没错，也可以理解，但是在ET模式下，为了避免数据饿死问题，用户态必须用一个循环，将所有的数据一次性处理结束．所以在ET模式下下，虽然epoll系统调用减少了，但是用户态的逻辑复杂了，write/read调用增多了．所以这不好判断，要看用户的性能瓶颈在哪．
---
layout: post
title: "Linux的select"
date: 2018-03-01 18:34
comments: false
tags: 
- IO
- Linux
categories:	
- 基础
- CS

---

关于这篇文章：
* 内容：Linux的 select 的API和它的实现。
* 目的：学习Java Nio的过程中，想了解一下 select
* 注意：部分内容来自网上，不敢保证一定对。

<!--more-->


### select 基础

#### API
```
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
           
// 将 fd 从集合移除
void FD_CLR(int fd, fd_set *set);

// 检查给定的 fd 是否在集合里面
int FD_ISSET(int fd, fd_set *set);

// 将 fd 添加到集合
void FD_SET(int fd, fd_set *set);

// 清空集合
void FD_ZERO(fd_set *set);           
```
##### 入参解析：
* nfds : 3个集合中 fd 数目的最大值减1。
* readfds ： 这个集合的 fd 将被检查是否 “可读”
* writefds ： 这个集合的 fd 将被检查是否 “可写”
* exceptfds ：这个集合的 fd 将被检查是否 “出错”
* timeout ： select 调用的最大阻塞时间（数据结构为 timeval）
  * 如果 timeval 的2个值都是 0 ，那么 select 将立即返回
  * 如果 timeval 是 NULL， 那么 select 可以永久阻塞
  * <font color=blue>timeval的数据结构</font>：
   ```
   struct timeval {
   long    tv_sec;         /* seconds */
   long    tv_usec;        /* microseconds */
   };
   ```

##### 返回结果：
* 如果成功，返回在这3个集合的 fd 的数量
* 如果 timeout，那么返回 0。
* 如果 error，那么返回 -1，并且 errno会被设置

##### 注意：
<font color=red>这3个集合是会被 select 修改的，用于指出哪一个 fd 是就绪的。</font>

#### 使用例子
```
fd_set fd_in, fd_out;
struct timeval tv;
 
// Reset the sets
FD_ZERO( &fd_in );
FD_ZERO( &fd_out );
 
// Monitor sock1 for input events
FD_SET( sock1, &fd_in );
 
// Monitor sock2 for output events
FD_SET( sock2, &fd_out );
 
// Find out which socket has the largest numeric value as select requires it
int largest_sock = sock1 > sock2 ? sock1 : sock2;
 
// Wait up to 10 seconds
tv.tv_sec = 10;
tv.tv_usec = 0;
 
// Call the select
int ret = select( largest_sock + 1, &fd_in, &fd_out, NULL, &tv );
 
// Check if select actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    if ( FD_ISSET( sock1, &fd_in ) )
        // input event on sock1
 
    if ( FD_ISSET( sock2, &fd_out ) )
        // output event on sock2
}
```

### select 的实现
select源码位于fs/select.c文件。

#### select 的调用链 :
```
-----------    -----------------    ------------
sys_select |-->| core_sys_select|-->| do_select |
-----------    -----------------    ------------
```

#### sys_select
这是用户态下select系统调用入口：
```
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,
        fd_set __user *, exp, struct timeval __user *, tvp)
{
    struct timespec end_time, *to = NULL;
    struct timeval tv;
    int ret;

    if (tvp) {
        if (copy_from_user(&tv, tvp, sizeof(tv)))
            return -EFAULT;

        to = &end_time;
        if (poll_select_set_timeout(to,
                tv.tv_sec + (tv.tv_usec / USEC_PER_SEC),
                (tv.tv_usec % USEC_PER_SEC) * NSEC_PER_USEC))   //微秒转纳秒
            return -EINVAL;
    }

    ret = core_sys_select(n, inp, outp, exp, to);
    ret = poll_select_copy_remaining(&end_time, tvp, 1, ret);

    return ret;
}
```
sys_select 函数的实现很简单, 从用户进程拷贝超时时间, 调用 core_sys_select 函数。内核里面的时间是通过 clock 来计算， 所以会先把用户时间转化为 clock 数。

#### core_sys_select
```
int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
			   fd_set __user *exp, struct timespec *end_time)
{
	fd_set_bits fds;
	void *bits;
	int ret, max_fds;
	unsigned int size;
	struct fdtable *fdt;
	/* Allocate small arguments on the stack to save memory and be faster */
	long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];
 
	ret = -EINVAL;
	if (n < 0)
		goto out_nofds;
 
	/* max_fds can increase, so grab it once to avoid race */
	rcu_read_lock();
	fdt = files_fdtable(current->files);
	max_fds = fdt->max_fds;
	rcu_read_unlock();
	if (n > max_fds)
		n = max_fds;
 
	/*
	 * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
	 * since we used fdset we need to allocate memory in units of
	 * long-words. 
	 */
	size = FDS_BYTES(n);
	bits = stack_fds;
	if (size > sizeof(stack_fds) / 6) {
		/* Not enough space in on-stack array; must use kmalloc */
		ret = -ENOMEM;
		bits = kmalloc(6 * size, GFP_KERNEL);
		if (!bits)
			goto out_nofds;
	}
	fds.in      = bits;
	fds.out     = bits +   size;
	fds.ex      = bits + 2*size;
	fds.res_in  = bits + 3*size;
	fds.res_out = bits + 4*size;
	fds.res_ex  = bits + 5*size;
 
	if ((ret = get_fd_set(n, inp, fds.in)) ||
	    (ret = get_fd_set(n, outp, fds.out)) ||
	    (ret = get_fd_set(n, exp, fds.ex)))
		goto out;
	zero_fd_set(n, fds.res_in);
	zero_fd_set(n, fds.res_out);
	zero_fd_set(n, fds.res_ex);
 
	ret = do_select(n, &fds, end_time);
 
	if (ret < 0)
		goto out;
	if (!ret) {
		ret = -ERESTARTNOHAND;
		if (signal_pending(current))
			goto out;
		ret = 0;
	}
 
	if (set_fd_set(n, inp, fds.res_in) ||
	    set_fd_set(n, outp, fds.res_out) ||
	    set_fd_set(n, exp, fds.res_ex))
		ret = -EFAULT;
 
out:
	if (bits != stack_fds)
		kfree(bits);
out_nofds:
	return ret;
}
```
以下是 core_sys_select 的工作流程：
* 准备工作：
 * 定义一个SELECT_STACK_ALLOC(256字节)大小的栈上数组用于高效处理传入以及待传出的可读、可写及异常文件描述符集合，空间可能不够使用。
 * 基于current宏检查传入的最大fd对应参数n是否超出当前进程打开的文件描述符表内所示位图容量的max_fds数值（位数），基于使用位图结构也就很容易理解为何select调用的第一个参数是传入的待监听fd的最大值加1。
 * 栈上数组空间不足以存放本次select要处理的fd集合所需总计内存，则使用kmalloc（基于slab）从内核空间分配所需的连续物理内存。
 * 依次使用get_fd_set拷贝待监听的可读、可写及异常事件对应的文件描述符集合，<font color=cream>可见每次select调用都需要从用户空间拷贝传入的文件描述符集合到内核空间</font>。
 * 清空(zero)待传出的处理结果对应的文件描述符集合，用来存放本次select调用的结果。
* 调用 `do_select`
* 收尾工作：
 * 依次使用set_fd_set拷贝各事件处理结果集合到对应传入的三个事件集合，<font color=cream>可见每次select调用还要将处理结果从内核空间拷贝回用户空间下</font>。
 * 因select返回时最终结果事件集合会拷贝到传入的各事件初始集合（实际在调用前的准备工作也对待传出结果集合进行了清空，原始待监听各集合不可作为下次调用时复用），<font color=cream>所以每次select调用前都需要清空（FD_ZERO）传入的fd事件集合</font>。
 * 如果传入的文件描述符比较大，超出栈上分配的内存导致从内核空间分配了所需内存，则释放该内核对应内存。


<font color=purle> 补充</font>：
存放文件描述符集合的类型是`fd_set`：
```
typedef __kernel_fd_set     fd_set;

#undef __NFDBITS
#define __NFDBITS   (8 * sizeof(unsigned long))

#undef __FD_SETSIZE
#define __FD_SETSIZE    1024

#undef __FDSET_LONGS
#define __FDSET_LONGS   (__FD_SETSIZE/__NFDBITS)

#undef __FDELT
#define __FDELT(d)  ((d) / __NFDBITS)

#undef __FDMASK
#define __FDMASK(d) (1UL << ((d) % __NFDBITS))

typedef struct {
    unsigned long fds_bits [__FDSET_LONGS];
} __kernel_fd_set;
```
可以看到 `fd_set` 类型在内核里是实际上是 `__kernel_fd_set` 结构体，里面只包含了一个 unsigned long 类型的数组 fds_bits。这个数组的大小是 1024/(8 * sizeof(unsigned long)) ，也就是这个数组占用空间为 1024bit 。在select中文件描述符在集合里是以位图的形式存在的，把文件描述符存放在三个集合中，最大直到 1023 ，也就是只能监听最多 1024 个文件描述符，并且只能是0 ~ 1023。

所以：<font color=red>**存放文件描述符的数据结构限制了 select() 最多只能监听 1024 个文件描述符**</font>。

#### do_select
```
/*do_select 
真正的select在此,遍历了所有的fd,调用对应的xxx_poll函数 
*/  
int do_select(int n, fd_set_bits *fds, s64 *timeout)  
{  
    struct poll_wqueues table;  
    poll_table *wait;  
    int retval, i;  
  
    rcu_read_lock();  
    /*根据已经打开fd的位图检查用户打开的fd, 要求对应fd必须打开, 并且返回最大的fd*/  
    retval = max_select_fd(n, fds);  
    rcu_read_unlock();  
  
    if (retval < 0)  
        return retval;  
    n = retval;  
  
  
    /*将当前进程放入自已的等待队列table, 并将该等待队列加入到该测试表wait*/  
    poll_initwait(&table);  
    wait = &table.pt;  
  
    if (!*timeout)  
        wait = NULL;  
    retval = 0;  
  
    for (;;) {/*死循环*/  
        unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;  
        long __timeout;  
  
        /*注意:可中断的睡眠状态*/  
        set_current_state(TASK_INTERRUPTIBLE);  
  
        inp = fds->in; outp = fds->out; exp = fds->ex;  
        rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;  
  
  
        for (i = 0; i < n; ++rinp, ++routp, ++rexp) {/*遍历所有fd*/  
            unsigned long in, out, ex, all_bits, bit = 1, mask, j;  
            unsigned long res_in = 0, res_out = 0, res_ex = 0;  
            const struct file_operations *f_op = NULL;  
            struct file *file = NULL;  
  
            in = *inp++; out = *outp++; ex = *exp++;  
            all_bits = in | out | ex;  
            if (all_bits == 0) {  
                /* 
                __NFDBITS定义为(8 * sizeof(unsigned long)),即long的位数。 
                因为一个long代表了__NFDBITS位，所以跳到下一个位图i要增加__NFDBITS 
                */  
                i += __NFDBITS;  
                continue;  
            }  
  
            for (j = 0; j < __NFDBITS; ++j, ++i, bit <<= 1) {  
                int fput_needed;  
                if (i >= n)  
                    break;  
  
                /*测试每一位*/  
                if (!(bit & all_bits))  
                    continue;  
  
                /*得到file结构指针，并增加引用计数字段f_count*/  
                file = fget_light(i, &fput_needed);  
                if (file) {  
                    f_op = file->f_op;  
                    mask = DEFAULT_POLLMASK;  
  
                    /*对于socket描述符,f_op->poll对应的函数是sock_poll 
                    注意第三个参数是等待队列，在poll成功后会将本进程唤醒执行*/  
                    if (f_op && f_op->poll)  
                        mask = (*f_op->poll)(file, retval ? NULL : wait);  
  
                    /*释放file结构指针，实际就是减小他的一个引用计数字段f_count*/  
                    fput_light(file, fput_needed);  
  
                    /*根据poll的结果设置状态,要返回select出来的fd数目，所以retval++。 
                    注意：retval是in out ex三个集合的总和*/  
                    if ((mask & POLLIN_SET) && (in & bit)) {  
                        res_in |= bit;  
                        retval++;  
                    }  
                    if ((mask & POLLOUT_SET) && (out & bit)) {  
                        res_out |= bit;  
                        retval++;  
                    }  
                    if ((mask & POLLEX_SET) && (ex & bit)) {  
                        res_ex |= bit;  
                        retval++;  
                    }  
                }  
  
                /* 
                注意前面的set_current_state(TASK_INTERRUPTIBLE); 
                因为已经进入TASK_INTERRUPTIBLE状态,所以cond_resched回调度其他进程来运行， 
                这里的目的纯粹是为了增加一个抢占点。被抢占后，由等待队列机制唤醒。 
 
                在支持抢占式调度的内核中（定义了CONFIG_PREEMPT），cond_resched是空操作 
                */   
                cond_resched();  
            }  
            /*根据poll的结果写回到输出位图里*/  
            if (res_in)  
                *rinp = res_in;  
            if (res_out)  
                *routp = res_out;  
            if (res_ex)  
                *rexp = res_ex;  
        }  
        wait = NULL;  
        if (retval || !*timeout || signal_pending(current))/*signal_pending前面说过了*/  
            break;  
        if(table.error) {  
            retval = table.error;  
            break;  
        }  
  
        if (*timeout < 0) {  
            /*无限等待*/  
            __timeout = MAX_SCHEDULE_TIMEOUT;  
        } else if (unlikely(*timeout >= (s64)MAX_SCHEDULE_TIMEOUT - 1)) {  
            /* 时间超过MAX_SCHEDULE_TIMEOUT,即schedule_timeout允许的最大值，用一个循环来不断减少超时值*/  
            __timeout = MAX_SCHEDULE_TIMEOUT - 1;  
            *timeout -= __timeout;  
        } else {  
            /*等待一段时间*/  
            __timeout = *timeout;  
            *timeout = 0;  
        }  
  
        /*TASK_INTERRUPTIBLE状态下，调用schedule_timeout的进程会在收到信号后重新得到调度的机会， 
        即schedule_timeout返回,并返回剩余的时钟周期数 
        */  
        __timeout = schedule_timeout(__timeout);  
        if (*timeout >= 0)  
            *timeout += __timeout;  
    }  
  
    /*设置为运行状态*/  
    __set_current_state(TASK_RUNNING);  
    /*清理等待队列*/  
    poll_freewait(&table);  
  
    return retval;  
}  
  
  
static unsigned int sock_poll(struct file *file, poll_table *wait)  
{  
    struct socket *sock;  
  
    /*约定socket的file->private_data字段放着对应的socket结构指针*/  
    sock = file->private_data;  
  
    /*对应了三个协议的函数tcp_poll, udp_poll, datagram_poll，其中 udp_poll几乎直接调用了 datagram_poll */  
    return sock->ops->poll(file, sock, wait);  
} 
```
上面代码的逻辑：
* select会循环遍历它所监测的 fd_set（一组文件描述符(fd)的集合）内的所有文件描述符对应的驱动程序的poll函数。
* <font color=red>驱动程序提供的poll函数首先会将调用select的用户进程插入到该设备驱动对应资源的等待队列(如读/写等待队列)，然后返回一个bitmask告诉select当前资源哪些可用</font>。
* 当select循环遍历完所有fd_set内指定的文件描述符对应的poll函数后，如果没有一个资源可用(即没有一个文件可供操作)，则select让该进程睡眠(通过使用 schedule_timeout 方法)，一直等到有资源可用为止，进程被唤醒(或者timeout)继续往下执行。

由上述可见，<font color=orange>每次select调用都要轮询完成所有fd的挂载等待队列及事件监测</font>。

**<font color=tan>补充</font>**：
唤醒这个调用 select 的进程通常是在所监测文件的设备驱动内实现的，驱动程序维护了针对自身资源读写的等待队列。<font color=red>当设备驱动发现自身资源变为可读写并且有进程睡眠在该资源的等待队列上时，就会唤醒这个资源等待队列上的进程</font>。



#### 总结
 1. 每次调用select，都需要把fd集合从用户态拷贝到内核态
 2. select 支持的文件描述符数量最大是1024
 3. select 为了能收到事件通知，还需要完成所有 fd 的挂载等待队列，当然还需要在 select 结束前清空等待队列。
 3. select 需要把所有的文件描述符都轮询一遍（依靠驱动程序的poll函数）才能知道那些 fd 是就绪的。
 4. 返回给用户进程时，它是把3个 fd_set 的结果拷贝到用户空间，让用户进程去遍历这3个集合。
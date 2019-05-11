---
layout: post
title: "Linux的mmap"
date: 2018-03-02 19:45
comments: false
tags: 
- IO
- Linux
categories:	
- 基础
- CS

---

关于这篇文章：
* 内容：Linux的 mmap 的原理。
* 目的：学习Java Nio的过程中，想了解一下 mmap
* 注意：部分内容来自网上，不敢保证一定对。

<!--more-->


### mmap的类型
* 文件映射 ：将文件的一个区间映射到进程的虚拟地址空间
* 匿名映射 ：进程的虚拟地址空间对应的页初始化为0.

### mmap的定义
> #include <sys/mman.h>
>
> void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
>
> Returns starting address of mapping on success, or MAP_FAILED on error

#### 参数解析
* addr ：表示这个创建的mapping被放在虚拟地址空间的哪里，一般设置为NULL，由内核选择合适的位置。
* length ：mapping的大小
* prot  ： 用于指明mapping的内存保护标志。
* flags ： 用于映射操作时的控制，有 MAP_PRIVATE、MAP_SHARED。
* fd ： 文件描述符，如果不是做文件映射，这个值被忽略。
* offset ： 表示从文件的该位置起，开始映射。如果不是做文件映射，这个值被忽略。


### mmap文件映射

#### 虚拟地址空间
<img width=300 src="/assets/blogImg/linux_mmap/1.png">

#### 内存描述符
<img width=700 src="/assets/blogImg/linux_mmap/2.png">

#### 文件描述符
<img width=600 src="/assets/blogImg/linux_mmap/3.png">

#### mmap文件映射的原理
mmap内存映射的实现过程，总的来说可以分为三个阶段：

##### 阶段1：进程启动映射过程，并在虚拟地址空间中为映射创建虚拟映射区域
1. 进程在用户空间调用库函数mmap。
2. 在当前进程的虚拟地址空间中，寻找一段空闲的满足要求的连续的虚拟地址
3. 为此虚拟区分配一个vm_area_struct结构，接着对这个结构的各个域进行了初始化
4. 将新建的虚拟区结构（vm_area_struct）插入进程的虚拟地址区域链表或树中


##### 阶段2：调用内核空间的系统调用函数mmap
<font color=orange>这里的mmap是内核函数，实现了文件物理地址和进程虚拟地址的一一映射关系</font>。
5. 为映射分配了新的虚拟地址区域后，通过待映射的文件指针，在文件描述符表中找到对应的文件描述符，通过文件描述符，链接到内核“已打开文件集”中该文件的文件结构体（struct file），每个文件结构体维护着和这个已打开文件相关各项信息。
6. 通过该文件的文件结构体，链接到file_operations模块，调用内核函数mmap，其原型为：`int mmap(struct file *filp, struct vm_area_struct *vma)`，不同于用户空间库函数。
7. 内核mmap函数通过虚拟文件系统inode模块定位到文件磁盘物理地址。
8. 通过remap_pfn_range函数建立页表，即实现了文件地址和虚拟地址区域的映射关系。此时，这片虚拟地址并没有任何数据关联到主存中。


##### 阶段3：进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存（主存）的拷贝
<font color=orange>前两个阶段仅在于创建虚拟区间并完成地址映射，但是并没有将任何文件数据的拷贝至主存。真正的文件读取是当进程发起读或写操作时</font>。
9. 进程的读或写操作访问虚拟地址空间这一段映射地址，通过查询页表，发现这一段地址并不在物理页面上。因为目前只建立了地址映射，真正的硬盘数据还没有拷贝到内存中，因此引发缺页异常。
10. 缺页异常进行一系列判断，确定无非法操作后，内核发起请求调页过程。
11. 调页过程先在交换缓存空间（swap cache）中寻找需要访问的内存页，如果没有则调用`nopage`函数把所缺的页从磁盘装入到主存中。
12. 之后进程即可对这片主存进行读或者写的操作，如果写操作改变了其内容，一定时间后系统会自动回写脏页面到对应磁盘地址，也即完成了写入到文件的过程。


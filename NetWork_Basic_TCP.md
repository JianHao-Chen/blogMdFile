---
layout: post
title: "TCP简介"
date: 2019-04-02 08:55
comments: false
tags: 
- 网络
categories:	
- 基础
- 网络
---

本文简单介绍TCP协议。

<!--more-->


### TCP首部
<img height=400 src="/assets/blogImg/NetWork_Basic/TCP/1.png">

其中，最常关注的字段有：
* Sequence Number：包的序号，用来解决网络包乱序（reordering）问题
* Acknowledgement Number：就是ACK——用于确认收到，用来解决不丢包的问题。
* Window：又叫Advertised-Window，也就是著名的滑动窗口（Sliding Window），用于解决流控的。
* TCP Flag：也就是包的类型，主要是用于操控TCP的状态机的。

### TCP三次握手
<img height=400 src="/assets/blogImg/NetWork_Basic/TCP/2.png">

#### 流程
* 第一次握手：
Client将标志位SYN置为1，随机产生一个值seq=x，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。
* 第二次握手：
  * Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack=x+1，随机产生一个值seq=y，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。
  * Client收到确认后，检查ack是否为x+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=y+1，并将该数据包发送给Server，Client进入ESTABLISHED状态。
* 第三次握手：
Server检查ack是否为y+1，ACK是否为1，如果正确则连接建立成功，Server进入ESTABLISHED状态。

到此，完成三次握手，随后Client与Server之间可以开始传输数据了。

#### <font color=red>为什么需要三次握手?</font>
* 分配资源
* 初始化并告诉peer端序列号

### TCP四次挥手
<img height=200 src="/assets/blogImg/NetWork_Basic/TCP/3.png">

#### 流程
* 第一次挥手: Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
* 第二次挥手: Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
* 第三次挥手: Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
* 第四次挥手：
  * Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，在等待 2MSL 之后，Client进入 CLOSED状态
  * Server收到ACK，进入CLOSED状态

到此完成四次挥手。

#### <font color=red>为什么需要四次挥手?</font>
其实断开连接是2次挥手，因为TCP是全双工，发送方和接收方都需要Fin和Ack。


### TCP的状态机
<img height=400 src="/assets/blogImg/NetWork_Basic/TCP/4.png">

### TCP的序号
一个TCP发送数据的例子：
```
       (SYN)                   Seq=0
------------------------>
       (SYN, ACK)              Seq=0, ack=1
<------------------------
       (ACK)                   Seq=1, ack=1
------------------------>
       (ACK)[len:1440]         Seq=1, ack=1
------------------------->
       (ACK)[len:1440]         Seq=1441, ack=1
------------------------->
       (ACK)                   Seq=1, ack=2881
<------------------------
```
注意到：
* Seq的值与传输的字节数有关的，实际上，Seq就是用于为每一个TCP数据包标序号。
* 接收方接收到数据包后，需要发送接收应答包，其中包含确认序号（ack=Sender Seq + 1），表示期待收到的下一个序号。
* SYN（FIN，这个例子没有）需要消耗一个序号
* 发送ACK不需任何代价


### TCP重传机制
TCP要保证所有的数据包都可以到达，所以，必需要有重传机制。

#### 超时重传机制
在发送时设置定时器，如果时间到还没有收到确认，就重传数据。

重传的策略有：
* 仅重传timeout的包。
* 重传timeout的包及其后所有的包

这两种方式有好也有不好。第一种会节省带宽，但是慢，第二种会快一点，但是会浪费带宽，也可能会有无用功。但总体来说都不好。因为都在等timeout，timeout可能会很长

#### 快速重传机制
TCP引入了一种叫 Fast Retransmit 的算法，<font color=green>**不以时间驱动，而以数据驱动重传**</font>。如果包没有连续到达，就ack最后那个可能被丢了的包，如果发送方连续收到3次相同的ack，就重传。Fast Retransmit的好处是不用等timeout了再重传。


### TCP滑动窗口

#### 目的
为了实现TCP的<font color=blue>**可靠传输**</font>的其中一种<font color=orange>**流量控制**</font>技术。

#### 定义
TCP头里有一个字段叫Window，又叫Advertised-Window，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据。于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来。

<img height=400 src="/assets/blogImg/NetWork_Basic/TCP/5.png">

#### Zero Window
当接收方通知发送方它的窗口大小为0，发送方就不发送数据了。发送方如何得知什么时候可以发送数据呢？

TCP使用了Zero Window Probe技术，缩写为ZWP，也就是说，发送端在窗口变成0后，会发ZWP的包给接收方，让接收方来ack他的Window尺寸，一般这个值会设置成3次，第次大约30-60秒（不同的实现可能会不一样）。如果3次过后还是0的话，有的TCP实现就会发RST把链接断了。

#### 糊涂窗口综合症（Silly Window Syndrome）
如果接收窗口的大小只有几个字节，发送方也会发送这几个字节。

这样就存在着效率问题了：以太网的MTU为1500字节，除去TCP+IP头的40个字节，真正的数据传输可以有1460。如果你的网络包可以塞满MTU，那么你可以用满整个带宽，如果不能，那么你就会浪费带宽。

因此，糊涂窗口综合症可以比喻为：本来可以坐200人的飞机里只坐了一两个人。

解决办法分为发送端和接收端：
* 接收端：如果收到的数据导致window size小于某个阈值，直接回ack(0)，将window关闭
* 发送端：使用著名的 Nagle 算法：等到 Window Size>=MSS 或者 收到之前发送数据的ack回包，才发数据，否则就攒数据

Nagle算法默认是打开的，所以，对于一些需要小包场景的程序，比如像telnet或ssh这样的交互性比较强的程序，你需要关闭这个算法。你可以在Socket设置TCP_NODELAY选项来关闭这个算法（关闭Nagle算法没有全局参数，需要根据每个应用自己的特点来关闭）


### TCP的拥塞处理（Congestion Handling）

#### 背景
TCP通过滑动窗口进行流量控制，但是TCP觉得这还不够，因为并不知道网络中间发生了什么。TCP的设计者觉得，一个伟大而牛逼的协议仅仅做到流控并不够，因为流控只是网络模型4层以上的事，TCP的还应该更聪明地知道整个网络上的事。

具体一点，我们知道TCP通过一个timer采样了RTT并计算RTO，但是，如果网络上的延时突然增加，那么，TCP对这个事做出的应对只有重传数据，但是，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，于是，这个情况就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的TCP连接都这么行事，那么马上就会形成“网络风暴”，TCP这个协议就会拖垮整个网络。这是一个灾难。

所以，TCP不能忽略网络上发生的事情，而无脑地一个劲地重发数据，对网络造成更大的伤害。对此TCP的设计理念是：<font color=purple>**TCP不是一个自私的协议，当拥塞发生的时候，要做自我牺牲。就像交通阻塞一样，每个车都应该把路让出来，而不要再去抢路了**</font>。

#### 拥塞控制的构成
拥塞控制主要是四个算法：
* 慢启动
* 拥塞避免
* 拥塞发生
* 快速恢复

#### 慢启动
慢启动算法如下：
* 为发送方增加了一个窗口：<font color=brown>拥塞窗口（congestion window）</font>，记为cwnd
* 连接建好的开始先初始化cwnd = 1，表明可以传一个MSS大小的数据
* 每当收到一个ACK，cwnd+1; 呈线性上升
* 每当过了一个RTT，cwnd = cwnd*2; 呈指数上升
* 还，就会进入“拥塞避免算法”

<font color=pink>**拥塞窗口是发送方使用的流量控制，滑动窗口是接收方使用的流量控制**</font>。

#### 拥塞避免算法（Congestion Avoidance）
有一个ssthresh（slow start threshold），是一个上限，当cwnd >= ssthresh时，就会进入拥塞避免算法，逻辑如下：
* 收到一个ACK时，cwnd = cwnd + 1/cwnd
* 当每过一个RTT时，cwnd = cwnd + 1

避免增长过快导致网络拥塞，慢慢的增加调整到网络的最佳值。很明显，是一个线性上升的算法。

#### 拥塞发生时的算法
有2种情况：
* 等到RTO超时，重传数据包。TCP认为这种情况太糟糕，反应也很强烈：
  * sshthresh =  cwnd /2
  * cwnd 重置为 1
  * 进入慢启动过程
* 收到3个duplicate ACK时就开启重传，而不用等到RTO超时：
  * cwnd = cwnd /2
  * sshthresh = cwnd
  * 进入快速恢复算法——Fast Recovery
  

#### 快速恢复算法 – Fast Recovery
快速恢复算法是认为，你还有3个Duplicated Acks说明网络也不那么糟糕，所以没有必要像RTO超时那么强烈。 注意，正如前面所说，进入Fast Recovery之前，cwnd 和 sshthresh已被更新：
* cwnd = cwnd /2
* sshthresh = cwnd

真正的Fast Recovery算法如下：
* cwnd = sshthresh  + 3 * MSS （3的意思是确认有3个数据包被收到了）
* 重传Duplicated ACKs指定的数据包
* 如果再收到 duplicated Acks，那么cwnd = cwnd +1
* 如果收到了新的Ack，那么，cwnd = sshthresh ，然后就进入了拥塞避免的算法了。

还有其他的快速恢复算法，略。


### 总结

#### 目的
TCP提供面向连接的、可靠的字节流服务。

#### 实现“面向连接”
TCP负责连接的管理工作：
* 建立：3次握手
* 断开：4次挥手
* 保持：保活探测

#### 实现“可靠传输”
* 将数据分成若干个报文段，通过序号和确认序号，TCP可以：
  * 知道哪些数据被成功接收
  * 解决报文乱序、重复问题
* TCP通过它首部和数据的“检验和”，如检测不通过则丢弃报文，防止接收到被破坏的数据
* TCP通过超时重传，解决报文丢失问题
* TCP提供流量控制：通过“滑动窗口”
* TCP还提供拥塞控制，有4种算法：
  * 慢启动
  * 拥塞避免
  * 拥塞发生
  * 快速恢复
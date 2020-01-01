---
title: MESI
categories:
  - Java知识
tags:
  - 并发编程
  - CPU缓存
date: 2019-10-19 23:42:09
---

# MESI

CPU运算速度很快，内存运算速度很慢，它们在运算速度上有着较大的差距，如果让CPU等待内存处理完数据再进行下一步操作，会造成CPU资源浪费，为了解决这个问题，CPU高速缓存应运而生。

<!-- more --> 

在多核CPU系统中，就会存在多个一级缓存，那么相同的数据就会出现副本，可能会造成缓存数据不一致，为了不让系统数据混乱。这里引入MESI来协议保证系统正常运行。

MESI是4种状态的首字母，它们分别是修改态（Modified）、独占态（Exclusive）、共享态（Shared）、无效态（Invalid）。代表了缓存中数据可能处于的状态。



### 双核读取

①CPU A发出了一条指令，从主内存中读取x：
    CPU A从主内存通过bus读取到 cache a中并将该cache line设置为E状态。
②CPU B发出了一条指令，从主内存中读取x：
    CPU B试图从主内存中读取x时，CPU A检测到了地址冲突，将x标记为S状态。此时x存储于cache a和cache b中，其状态都为S。

{% asset_img 1.png 双核读取 %} 



### 修改数据

①CPU A发指令修改x：
    CPU A将x设置为M状态并通知缓存了x的CPU B，CPU B将cache b中的x设置为I状态。
    CPU A对x进行赋值。

{% asset_img 2.png 修改数据 %} 



### 同步数据

①CPU B发出指令读取x：
    CPU B通知CPU A，CPU A将修改后的数据同步到主内存和CPU B的cache，x的状态变为S。

{% asset_img 3.png 同步数据 %} 



MESI的状态转移中有两个需要特别注意的地方：

1.**I-->M，S-->M**。当CPU A对S状态的数据进行写操作时，需要在总线上发送invalidate请求，并等待其他CPU的invalidate acknowledge。同样地，对I状态的数据进行写操作，需要发送read invalidate请求并等待结果。这会降低CPU A的性能。

2.**MES-->I**。CPU A发送invalidate请求给CPU B，CPU B收到请求后，并不能立即回复invalidate acknowledge，主要原因是invalidate cacheline的操作没有那么快完成，特别是cache比较繁忙的时候。这时，CPU往往进行密集的loading和storing的操作，而来自其他CPU的，对本CPU local cacheline的操作需要和本CPU密集的cache操作进行竞争，只要完成了invalidate操作之后，本CPU才会发送invalidate acknowledge。



针对以上两个地方，MESI协议使用Store Buffer和Invalidate Queue来优化性能。

{% asset_img 4.png Store Buffer与Invalidate Queue %} 



#### Store Buffer

一旦增加了Store Buffer，那么CPU A无需等待其他CPU的响应，只需要将要修改的内容放入Store Buffer，然后就可以继续执行。当cacheline完成bus transaction后，就可以将修改的数据将从Store Buffer写进cacheline。



##### 引入Store Buffer之后的问题

1. CPU A运行foo()，CPU B运行bar()，a在CPU A中，b在CPU B中。

2. CPU A将a=1写进Store Buffer，发送read invalidate请求，然后执行b=1。

3. CPU B通过read请求读到b=1，循环。

4. 在CPU B这端，a的值为0，并不是预期的1。这时invalidate请求还未到达CPU B，a不会变成I状态。

具体过程可参考文献。

```c
//CPU A
//cacheline b
void foo()
{
    a = 1;
    b = 1;
}

//CPU B
//cacheline a
void bar()
{
    while (b == 0) continue;
    assert(a == 1);
}
```





#### Invalidate Queues

CPU在响应invalidate请求时，其实不需要完成invalidate操作就可以回送acknowledgement消息。CPU可以将invalidate message放入Invalidate Queues，然后直接回应acknowledgement，表示自己已经收到请求，随后会慢慢处理。当然，再慢也要有一个度，例如在对x变量的invalidate处理完成之前，不会发送有关x的请求出去。





##### 引入Invalidate Queues之后的问题

与Store Buffer所带来的问题类似。在CPU B这端，invalidate操作并没有被真正执行，所以变量a的值为0且状态有效，并不会得到预期的1.



### 内存屏障

硬件工程师为了将CPU性能发挥到极致，引入了Store Buffer和Invalidate Queues，那么相应地也带来了可见性和有序性的问题。于是硬件工程师又加入了Memory Barrier指令来解决上述问题。

看到这里，可能会有一个问题：硬件为什么不把这些底层的信息屏蔽掉呢，帮我们自动加上Memory Barrier就可以了？回答是不能的，因为软件经过编译处理变成硬件指令，硬件是不知道顶层业务逻辑的。



#### 写屏障

CPU收到写Memory Barrier指令，要等到收到invalidate ack，并将store buffer中所有数据写到cacheline之后才会执行后续指令。



### 读屏障

CPU收到读Memory Barrier指令，要等到执行完Invalidate Queue中的所有invalidate命令之后才会执行后续指令。

```c
//CPU A
//cacheline b
void foo()
{
    a = 1;
    //写屏障
    b = 1;
}

//CPU B
//cacheline a
void bar()
{
    while (b == 0) continue;
    //读屏障
    assert(a == 1);
}
```



## 参考资料

1. MESI：https://www.cnblogs.com/yanlong300/p/8986041.html

2. Linux内核同步机制之（三）: http://www.wowotech.net/kernel_synchronization/memory-barrier.html

3. 《Memory Barriers: a Hardware View for Software Hackers》

4. 《Memory Barriers: a Hardware View for Software Hackers》翻译：http://www.wowotech.net/kernel_synchronization/Why-Memory-Barriers.html
5. 《深入理解并行编程》作者Paul E. McKenney：http://www2.rdrop.com/users/paulmck/








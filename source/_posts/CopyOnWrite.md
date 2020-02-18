---
title: CopyOnWrite
categories:
  - 并发设计模式
tags:
  - CopyOnWrite
  - CopyOnWriteArrayList
  - fork
  - Redis
date: 2020-01-04 23:05:39
---

写时复制（Copy On Write，CoW）是一种计算机程序设计领域的优化策略。核心思想是，多个调用者同时请求相同资源时，他们会共同获得相同的指针指向相同的资源。直到某个调用者试图修改资源的内容时，系统才会真正复制一份给该调用者只读同一块资源，而其他调用者所见到的最初的资源仍然保持不变。**之后根据策略的不同，复制出的资源可以覆盖旧资源，也可以仅供写资源的调用者使用**。

<!-- more --> 

# Linux中的CoW

在类Unix操作系统中创建进程的API是`fork()`，传统的fork()函数会创建父进程的一个完整副本，例如父进程的地址空间现在用到了1G的内存，那么子进程就要复制父进程整个1G地址空间，这个过程是很耗时的。

在Linux中，fork()子进程并不复制整个进程的地址空间，而是让父子进程共享同一个地址空间。而后只在父进程或者子进程需要写入的时候（如调用`exec()`系列函数，该函数会装载一个新的程序，替换掉当前进程的地址空间），才会复制地址空间，从而使父子进程拥有各自的地址空间。

fork()之后，内核把父进程中所有内存页的权限设为read-only，然后子进程的地址空间指向父进程。当父子进程都只读内存时，相安无事。当其中某个进程写内存时，CPU检测到内存页是read-only，于是触发页异常中断（page-fault），陷入内核的一个中断例程。中断例程中，内核会把触发异常的页复制一份，于是父子进程各自持有独立的一页。



# Redis中的CoW

Redis的哈希表在rehash时，会根据BGSAVE命令或BGREWRITEAOF命令是否正在执行，决定要不要提高负载因子。

这是因为在执行上述命令的过程中，Redis会执行`fork()`创建子进程，而大多数操作系统都采用Copy on Write技术来优化子进程的使用效率，所以在子进程存在期间，Redis会提高负载因子，从而尽可能地避免在此期间进行哈希表扩容。这可以避免不必要的内存写入操作，避免不必要的资源复制。



# Java中的CoW

并发包中的CopyOnWriteArrayList和CopyOnWriteArraySet使用了CoW的思想实现了线程安全的List。相比较于对所有方法加了锁的Vector来说，CoW性能更好。CopyOnWriteArrayList对于读不加锁，性能很好；对于写加锁。

CopyOnWriteArrayList也有一些限制：

1. 每次写操作需要创建新的数组，容量较大时易频繁GC。
2. 数据实时性不高，读取数据时，可能会拿到旧的数组。

因此CopyOnWriteArrayList比较适合读多写少且对一致性要求不高的场景。例如RPC框架中保存在客户端的路由表。

```java
//CopyOnWriteArrayList.java
public E get(int index) {
    return get(getArray(), index);
}

private E get(Object[] a, int index) {
    return (E) a[index];
}

public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```



# 总结

Linux中的fork()和Java中的CopyOnWriteArrayList都使用了写时复制（Copy on Write， CoW）的思想。fork()在写时会复制出两份资源，而CopyOnWriteArrayList会将新复制出的资源覆盖掉旧资源。

CoW的缺点是会消耗较多内存，所以在写操作频繁以及数据量较大时不适合使用。
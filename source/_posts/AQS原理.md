---
title: AQS原理
categories:
  - JDK
tags:
  - 并发编程
  - JDK源码
date: 2019-10-13 22:21:14
---

# AQS

## AQS的设计

AQS（AbstractQueuedSynchronizer）是JDK1.5并发包引入的一个小型框架。它为并发包中的许多同步器提供了底层实现。

 <!-- more --> 

同步器的基本思路很简单，它有两个基本操作：获取锁和释放锁。

```java
获取锁(acquire):
while (synchronization state does not allow acquire) {
	enqueue current thread if not already queued;
	possibly block current thread;
}
dequeue current thread if it was queued;
```

```java
释放锁(release):
update synchronization state;
if (state may permit a blocked thread to acquire)
	unblock one or more queued threads;
```



<embed src="AQS_queue.svg" width="100%" type="image/svg+xml"/>

AQS需要以下三个重要的部分协调工作：

### 同步状态

AQS类中有一个int型的变量，该变量是volatile的，保证了同步状态在线程间的可见性。AQS提供了三个方法来操作同步状态：getState、setState、compareAndSetState。



### 线程阻塞

AQS使用LockSupport.park阻塞线程，使用LockSupport.unpark解锁线程。有关LockSupport的详情，具体可参考这里。



### 线程队列

队列是AQS的核心，该队列是FIFO队列（FIFO队列不支持对线程进行优先级排序），用于维护处于等待锁状态的线程。关于队列，有CLH队列和MCS队列，两种队列的详情可参考这里。因为CLH队列能比较好地处理timeout和cancel线程这两种情况，因此AQS以CLH队列为基础，对其进行了改造。

CLH队列在刚被提出时，是用于自旋锁的，而在AQS中，它不再用于自旋锁。AQS对CLH做的最大的修改是新增了指向后继节点的变量。因为AQS在线程释放锁时，需要显式地去唤醒下一个在等待的线程。

修改后的队列就变成了双向链表。在不加锁的情况下，无法原子性地操作双向链表的前、后节点，那么可能会存在next指向null的情况，这个时候就需要从tail往前找。



## 如何使用

在AQS中，有以下方法可以被重写，且重写这些方法是唯一使用AQS的方法。AQS是一个抽象类，但其中没有抽象方法，它作为抽象类是希望有子类去继承它并重写以下几个方法。以下几个方法都不是抽象方法，如果把他们都定为抽象方法的话，每个子类都需要重写全部5个方法，但对于某些锁而言，只要重写其中tryAcquire和tryRelease两个方法即可。

- tryAcquire
- tryRelease
- tryAcquireShared
- tryReleaseShared
- isHeldExclusively

除了以上几个方法外，AQS中其他都是final方法，表示不能被子类修改。这么设计的原因是，AQS是一个整体性的框架，只修改其中某个方法没有意义，会使得整个框架无法按照预期工作。

下面是一个使用AQS实现互斥锁Mutex的例子。在类中新建内部类Sync继承AQS，这是一种比较建议的方法。这样的话，不会把AQS的细节对外暴露，而且对外暴露的公有方法可以取合适的名字。比如CountDownLatch的await()、ReentrantLock的lock()、Semaphore的acquire()。

tryAcquire:当能获取同步状态时，该方法需返回true

tryRelease:当新的同步状态可以被别的线程获取，该方法需返回true

```java
class Mutex {
    class Sync extends AbstractQueuedSynchronizer {
        public boolean tryAcquire(int ignore) {
            return compareAndSetState(0, 1);
        }
        public boolean tryRelease(int ignore) {
            setState(0); return true;
        }
    }
 
    private final Sync sync = new Sync();
    public void lock() { sync.acquire(0); }
    public void unlock() { sync.release(0); }
}
```



# Reference

《The java.util.concurrent Synchronizer Framework》
---
title: AQS应用
categories:
  - JDK
tags:
  - 并发编程
  - JDK源码
date: 2019-12-10 16:12:21
---

# AQS的应用

 <!-- more --> 

## ReentrantLock

线程可重入锁，与synchronized的作用相同，但两者特性不同。

ReentrantLock可配置为公平锁和非公平锁。多个线程一起竞争AQS对象中的state变量。若state为0，说明锁目前空闲可用，需用CAS竞争锁；若state不为0，说明有线程占用着锁，需要将本线程加入队列，并挂起本线程，等待唤醒。



## Semaphore

Semaphore用来控制并发数量，当数量为1时，Semaphore就退化为普通的锁。Semaphore不可重入。

AQS中的state变量表示允许的并发数。

```java
Semaphore semaphore = new Semaphore(10);
public void run() {
    try {
        semaphore.acquire();
        //do something，控制该资源访问并发数为10
    } catch (Exception e) {
        e.printStackTrace();
    }
    semaphore.release();
}
```



## CountDownLatch

CountDownLatch可以让线程等待某个其他线程各自执行完毕后再执行。

AQS中的state变量表示需要等待的数量。

```java
final CountDownLatch latch = new CountDownLatch(1);

//线程1
public void run() {
    //do something
    latch.countDown();
}

//线程2
public void run() {
    try {
        latch.await();//需等待线程1完成之后再继续
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
}

```





## ReentrantReadWriteLock
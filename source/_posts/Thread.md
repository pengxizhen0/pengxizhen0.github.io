---
title: Thread
categories:
  - Java知识
tags:
  - 并发编程
  - JDK源码
date: 2019-12-20 10:49:31
---

Java线程共有6种状态，在Thread类中有枚举来详细定义和说明：

<!-- more --> 

```java
//Thread.java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

- **NEW** 线程刚创建，尚未启动；

- **RUNNABLE** 线程正在正常运行中；

- **BLOCKED** 线程等待进入临界区，如等待另一个线程走出synchronized块；

- **WAITING** 线程在临界区里面等待唤醒，在临界区内调用wait()后进入WAITING状态，等待另一个调用notify()/notifyAll()；

- **TIMED_WAITING** 这个状态就是有时间限制的WAITING，如调用wait(long)、join(long)、sleep(long)；

- **TERMINATED** 线程执行完run()方法，处于该状态的线程会被GC。
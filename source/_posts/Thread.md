---
title: Thread
categories:
  - Java知识
tags:
  - 并发编程
  - JDK源码
date: 2019-12-20 10:49:31
---

# 通用的线程生命周期

通用的线程生命周期可以用五态模型来描述。这五态分别是：初始状态、可运行状态、运行状态、休眠状态和终止状态。 

```java
//五态模型
//
//  初始状态
//     ↓
//  可运行状态←---
//    ↑↓        ↑
//  运行状态  →	休眠状态
//     ↓
//  终止状态
```

1. **初始状态**。线程已经被创建，但是还不允许分配CPU执行。这个状态属于编程语言特有的，在操作系统层面，真正的线程还没有创建。
2. **可运行状态**。真正的操作系统线程已经成功创建，线程可以分配 CPU 执行。
3. **运行状态**。当有空闲的CPU时，操作系统会将其分配给一个处于可运行状态的线程，被分配到CPU的线程为、运行状态。
4. **休眠状态**。运行状态的线程如果调用一个阻塞API，那么线程的状态就会转换到休眠状态。
5. **终止状态**。线程执行完或者出现异常就会进入终止状态，终止状态的线程不会切换到其他任何状态，进入终止状态也就意味着线程的生命周期结束了。 

在不同的编程语言中，线程状态也不同。在Java中，线程共有6种状态，合并了可运行和运行状态，细化了休眠状态。在Thread类中有枚举来详细定义和说明：

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

- **NEW** 线程刚创建，还未分配CPU执行；

- **RUNNABLE** 线程可以分配CPU执行，调用`Thread.start()`可进入该状态；

- **BLOCKED** 线程等待进入临界区，只有等待synchronized隐式锁这一个场景，才会使线程进入该状态；

- **WAITING** 线程在临界区里面等待唤醒，调用`Object.wait()`、`Thread.join()`、`LockSupport.park()`这三个方法会使线程进入该状态；

- **TIMED_WAITING** 这个状态就是有时间限制的WAITING，调用带时间参数的`Objec.wait()`、`Thread.join()`、`LockSupport.park()`、`Thread.sleep()`这些方法会使线程进入该状态；

- **TERMINATED** 线程执行完run()方法或者抛出异常，处于该状态的线程会被GC。

```java
//Java线程模型
//
//  初始状态(NEW)
//     ↓
//  可运行/运行状态(RUNNABLE)←---→休眠状态(BLOCKED/WAITING/TIMED_WAITING)
//     ↓
//  终止状态(TERMINATED)
```



使用jstack诊断问题（待补充）

死锁、饥饿等情况可以通过跟踪分析线程的状态来解决
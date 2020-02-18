---
title: ThreadLocal
categories:
  - Java知识
tags:
  - 并发编程
  - JDK源码
date: 2019-09-09 10:06:25
---

# ThreadLocal

<!-- more --> 

ThreadLocal类提供了thread-local的变量，即每个线程都持有一份各自独立的变量。每个Thread类中有一个ThreadLocalMap类型的变量。

具体的ThreadLocalMap实例并不是ThreadLocal持有的，而是每个Thread持有，且不同的Thread持有不同的ThreadLocalMap实例, 因此它们不存在竞争。每个ThreadLocal只存一个数据，要在同个线程中存多个数据需要多个ThreadLocal。

下面是ThreadLocal对象引用图，下面的内容参考这张图看会更加容易理解。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
    	map.set(this, value);
    else
    	createMap(t, value);
}
```

建议使用static的ThreadLocal。既然是给整个线程使用的，那么将ThreadLocal定义成static就比较合理。

当使用局部变量结束后，ref指向ThreadLocal这个引用会消失，相应Entry的key会被回收。如果我们要再次使用ThreadLocal，那么就会new一个Entry。旧的Entry会在后续的操作中被回收。

ThreadLocal典型的应用是在web上。对于一个用户请求，web服务器会使用一个线程来服务这个请求。在web服务器中，线程的使用方式一般是线程池。我们一般可以在ThreadLocal中保存该用户的相关信息。



## 为什么使用弱引用

ThreadLocalMap中的key是弱引用，value是强引用。使用弱引用的原因是提高内存利用率，防止内存泄露（只能防止特定情况下的内存泄露）。

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    //...
}
```

如果key和value都是强引用的话，那么在整个线程活动中，Map中的对象是一直存在的，不能回收。换成弱引用的话，如果ThreadLocal ref这个强引用（如方法中的局部变量）消失的话，那么Map中的key就会被回收，key变为null。在后续对ThreadLocal的操作中（set/get/remove），ThreadLocal会检查key为null的Entry，并清除该Entry。但是，如果后续不对ThreadLocal进行操作，那个Entry中的value就会一直滞留，造成内存泄露。

如果使用了static的ThreadLocal，那么这条强引用就会一直存在，Entry中的弱引用就不会产生作用。



## 内存泄露

### 线程池的场景

**在线程池场景下，必须使用remove**。在该场景下，线程使用完毕并不会销毁，而是会回到池中等待下一次使用。如果在每次使用都set新对象的话，那么在Map中存储的值就会越来越多。由于指向Entry中的value的引用链一直是可到达的，旧的value不能回收，新的value不断往里放，导致内存泄露。

### 非线程池的场景

**在非线程池的场景下，可以不使用remove**。在该场景下，线程使用完毕会销毁，指向Entry的引用链不可达，这条链上的对象都会被回收。



<embed src="ThreadLocal.svg" width="100%" type="image/svg+xml"/>
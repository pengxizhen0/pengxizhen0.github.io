---
title: CAS
categories:
  - Java知识
tags:
  - 并发编程
  - JDK源码
date: 2019-10-16 16:29:52
---

# CAS

比较并交换(CompareAndSwap, CAS)是原子性更新变量的一种方式。Java中将对变量的原子性操作封装成原子类。

<!-- more --> 

```java
//AtomicInteger.java
public final int getAndSet(int newValue) {
    //valueOffset是偏移地址
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}

//Unsafe.class
public final int getAndSetInt(Object var1, long var2, int var4) {
    int var5;
    do {
        //获取旧值var5,compareAndSwapInt是个native方法
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var4));

    return var5;
}
```

那么`this.compareAndSwapInt(var1, var2, var5, var4)`再往下是什么呢？

最终在这个目录（`/hotspot/src/os_cpu/linux_x86/vm/atomic_linux_x86.inline.hpp`）可以看到compareAndSwapInt使用了CPU的底层指令。以下是linux_x86平台的代码。

```cpp
// 如果是多核处理器的环境，需要在指令前加lock前缀
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "

inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

对于Intel平台，CPU使用cmpxchgl指令来完成该功能，但它并不是原子的。在多核环境下，需要加lock指令来保证原子性。那么lock指令会产生什么动作呢。在老CPU上面（486 < CPU < P6），Lock指令会锁总线，这样来自其他处理器的请求会被阻塞；在新CPU上面（CPU > P6），Lock指令不会锁总线，而是会使用缓存一致性协议（MESI）来保证原子性。



## 参考文献

Intel开发手册：https://software.intel.com/en-us/articles/intel-sdm
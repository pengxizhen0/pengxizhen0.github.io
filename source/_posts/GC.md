---
title: GC
categories:
  - JVM
tags:
  - JVM
  - 垃圾回收
date: 2019-12-05 23:31:43
---

 

# GC

<!-- more --> 

## 回收谁

堆



## 如何判断是否要回收

从GC root出发，运用可达性分析，若访问不到的对象需要被回收。

GC root：

1. 本地变量引用的对象

2. 静态变量引用的对象

3. 常量引用的对象

4. Native方法引用的对象



对于弱引用，GC不会当它为引用，会进行回收



新生代使用复制算法

分为一个Eden区和两个Survivor区，大小为8:1:1。两个Survivor分别为From Survivor和To Survivor，每次GC过后，From和To会交换。新对象都出生在Eden区。







Partial GC：并不收集整个GC堆的模式

- Young GC：只收集young gen的GC
- Old GC：只收集old gen的GC。只有CMS的concurrent collection是这个模式
- Mixed GC：收集整个young gen以及部分old gen的GC。只有G1有这个模式

Full GC：收集整个堆，包括young gen、old gen、perm gen（如果存在的话）等所有部分的模式。



CMS：

初始标记（STW）单线程

并发标记

重新标记（STW）多线程

清除
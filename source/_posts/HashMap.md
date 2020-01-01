---
title: HashMap
categories:
  - Java知识
tags:
  - 并发编程
  - JDK源码
date: 2019-11-13 14:23:59
---

# HashMap

<!-- more-->

equals()相同的两个对象，hashCode()一定要相同。

equals()不同的两个对象，hashCode()可以相同也可以不同。

Object的hashCode()默认实现是将Object的地址转换为一个int变量。

Object的equals()默认实现是比较两个对象是否指的是一个对象。



比较两个对象时，可以先粗略地用hashCode进行比较，若不一致，则返回结果；若一致，则用==或者equals继续比较。



## put

1. 对key的原始hash值进行处理，在低位bit上加入高位bit信息，目的是减少小容量map的冲突。
2. 根据table的长度，对处理后的hash值截取低位bit。

3. 若该table槽为null，新增节点放入table；若该table槽不为null，则表明有冲突，HashMap采用拉链法解决冲突。



## size

HashMap有个成员变量size，求size的时间复杂度是O(1)。HashMap在put时size+1，在remove时size-1。



## ConcurrentModificationException

在遍历HashMap的过程中，如果进行了写操作，例如put, remove, clear等。HashMap会报并发修改错误。

这个是通过modCount变量实现的。在每个写操作中，modCount都会+1，表明经过了一次修改。在遍历开始时，当前的modCount会被记录下来记作m，每次遍历到一个元素时会检查当前modCount是否等于m，不相等说明发生了并发修改。



## Java 7

### 非线程安全

大家都知道HashMap是非线程安全的，在多线程情况下使用会出现问题，但具体会出现什么问题呢。下面我们来看一下多线程下，HashMap会出现的两个比较典型的问题：死循环和数据丢失。其实这两个问题并不是全部，由于多线程下，HashMap在各个地方都会出现问题，问题多多，所以这里就拿这两个例子来说明。



#### 链表成环

当两个线程都在进程扩容时，容易发生链表成环。线程A运行过语句`Entry<K,V> next = e.next;`之后就挂起。此时**e和next在一个链表上且顺序相反**。CPU开始运行线程B，线程B完成扩容后将CPU让出（图1）。图2\图3\图4去掉了与线程B相关的东西，专注在线程A的行为上。它们演示了经过3个循环，链表最终成环。



#### 数据丢失

我们调换一下原始节点在链表中的顺序，使得扩容后**e和next分布在两个链表上**，这样的话，e之前的节点都会丢失。



## Java 8的改进

### 链表优化为树

java8对table[]后面的链表进行了优化，当链表中节点个数≥8的时候，链表就会变为红黑树。当节点个数小于等于6的时候，红黑树又会变为链表。

- 为什么是8：当节点个数小于8的时候，链表遍历的平均时间复杂度(n/2，n=8时，需遍历4个节点)高于红黑树(O(logn)，n=8时，需遍历3个节点）。
- 为什么是6：为防止节点个数频繁变化导致的链表与红黑树相互转化，该值不能设置为8，设置为6比较合适。



### 对原始hash值的处理

在java8中，对原始hash值的处理相对简单很多。较为简单的处理会导致冲突加剧，即截取低位bit后的节点会分布不均匀。那么较为严重的冲突所带来的查找效率低下可以被红黑树所缓解一些。

```java
//java7
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }
    h ^= k.hashCode();
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

//java8
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```



### rehash逻辑变化

在java7中，扩容时rehash，采用`hash & (length-1)`计算出新的数组下标，该值是绝对值。

```java
//java7扩容rehash
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}

static int indexFor(int h, int length) {
    return h & (length-1);
}
```



在java8中，采用`hash & oldCap`计算出新的数组下标的相对值。rehash其实就是在原有hash基础上往高位再多取1bit。该bit位为0，rehash后数组下标不变；该bit位为1，rehash后数组下标需要加上原数组的大小。

```java
//java8扩容rehash（链表情况下）
do {
    next = e.next;
    //rehash后节点所在的数组下标不变
    if ((e.hash & oldCap) == 0) {
        if (loTail == null)
            loHead = e;
        else
            loTail.next = e;
        loTail = e;
    }
    //rehash后节点所在的数组下标需要加上原数组长度
    else {
        if (hiTail == null)
            hiHead = e;
        else
            hiTail.next = e;
        hiTail = e;
    }
} while ((e = next) != null);
if (loTail != null) {
    loTail.next = null;
    newTab[j] = loHead;
}
if (hiTail != null) {
    hiTail.next = null;
    newTab[j + oldCap] = hiHead;
}
```



### 扩容时倒序优化为顺序

在多线程对HashMap进行扩容时，链表rehash到新的数组后，顺序与原链表相反。这样子会造成链表成环，get()元素的时候进入死循环。HashMap是非线程安全的，虽然它的实现没有问题，是使用者错误使用了HashMap，但作者还是对这个点进行了优化，这个优化减少了错误使用HashMap时的问题。   



<embed src="java7_HashMap.svg" width="100%" type="image/svg+xml"/>

<embed src="java7_HashMap_loop.svg" width="100%" type="image/svg+xml"/>
<embed src="java7_HashMap_miss.svg" width="100%" type="image/svg+xml"/>
<embed src="java8_HashMap.svg" width="100%" type="image/svg+xml"/>
<embed src="扩容.svg" width="100%" type="image/svg+xml"/>

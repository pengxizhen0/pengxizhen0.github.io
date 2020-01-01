---
title: ConcurrentHashMap
categories:
  - JDK
tags:
  - 并发编程
  - JDK源码
date: 2019-11-12 19:18:28
---

# ConcurrentHashMap



## java7

java7中的ConcurrentHashMap主要使用分段锁保证线程安全。只能有一个线程对segment中的table[]进行写操作，读操作不加锁。

<!-- more -->

<embed src="java7_ConcurrentHashMap.svg" width="100%" type="image/svg+xml"/>
### put

1. 对原始hash值进行处理，目的是减少冲突
2. 取处理之后的hash值最高几位当做segment数组下标
3. 用CAS对segment进行初始化
4. 使用非阻塞的方式对segment尝试加锁，成功加锁则可以往里写；加锁失败则等待释放锁
5. 插入链表头部

```java
//ConcurrentHashMap的put方法
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);//处理原始hash值
    int j = (hash >>> segmentShift) & segmentMask;//计算segment的数组下标
    if ((s = (Segment<K,V>)UNSAFE.getObject
         (segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);//初始化segment
    return s.put(key, hash, value, false);
}

//segment的put方法
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    HashEntry<K,V> node = tryLock() ? null :
    scanAndLockForPut(key, hash, value);//写操作前对segment加锁
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            //需要遍历链表确认是否有相同的key
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            //若无相同的key，则将新节点插入链表头部
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```





```java
//返回相应数组下标的segment，若不存在则新建
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE;
    Segment<K,V> seg;
    //检查segment是否已存在
    //后续还会对这一点进行多次检查。在每次new之前都会检查一遍，目的是防止资源浪费
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        Segment<K,V> proto = ss[0];//segment[0]在最开始已经初始化
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        //再次检查
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) {
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            //while搭配CAS使用，只允许一个线程新建segment
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```



```java
//尝试加锁失败，则等待释放锁，在等待时还可以提前先把Node创建起来
//用自旋的方式等待，自旋一定次数后，若还没释放，则挂起线程等待
//若自旋时发现链表头结点发生变化，则重新遍历链表并重新自旋????????为什么这么设计
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```



### size

遍历segment，把每个segment中的数量累加起来。这样的遍历做两次，对比这两次的modCount结果是否一致（任何一个写操作都会使得modCount变量+1，modCount只增不减，类似于版本号）。时间复杂度为O(n)，n为segment的数量。

- 若一致，说明没有并发操作，这个size准确。
- 若不一致，说明有并发写操作，需要做第三次的遍历。第三次的modCount结果若与第二次相同，则返回size。若不相同，说明有并发，此时会获取全部segment的锁，再计算size。这时是进行不了任何写操作的。

```java
public int size() {
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
            //循环3遍，都有并发干扰计算结果，此时需要锁全部segment
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock();
            }
            sum = 0L;
            size = 0;
            overflow = false;
            //遍历全部segment，累加每个segment的size
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
            //modCount不变，说明无并发干扰，可以返回结果
            if (sum == last)
                break;
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```





## java8

java8中的ConcurrentHashMap去掉了分段锁，采用了粒度更细的锁。写操作对单个链表或树进行加锁。

<embed src="java8_ConcurrentHashMap.svg" width="100%" type="image/svg+xml"/>
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());//对原始hash值进行处理
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //如果table数组中该槽为空，则使用CAS新增
        //新增成功则返回；新增失败需要重新进入循环
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;
        }
        //如果处于扩容状态，该线程会帮忙进行扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //已经存在链表或树，需要锁住头节点
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```





### size

java8推荐使用mappingCount()方法，用于代替size()。mappingCount()返回long型，范围比size()的int型要大。

如果ConcurrentHashMap在非并发环境下进行写操作，那么元素的计数结果都会体现在baseCount成员变量上。如果出现并发，那么就会将计数结果更新到CounterCell[]数组，每个并发线程都在数组里拥有一个自己的槽。所以整个Map的元素数量就是baseCount+CounterCell[]数组中的数。

需要注意的是，mappingCount()和size()计算出来的是近似数，可能在过程中，还有线程在累加CounterCell[]，这个结果未能反映到最终结果中。

而java7中ConcurrentHashMap的size()求得的结果就不是近似值，是准确值。因为它能保证在方法的“调用时刻”和“返回时刻”之间，无写操作进入。

```java
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}

//与LongAdder原理类似
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

由于java8的ConcurrentHashMap的size()计算与LongAdder原理类似，这里就简单介绍一下LongAdder。

LongAdder是java8并发包中新引入的原子类，解决的是AtomicLong在高并发情况下的性能问题。原子类中有个value的成员变量，对value的CAS操作是一个自旋锁，在冲突严重的情况下，该锁会运行较长时间。

LongAdder的基本思路就是**分散热点**，将value值分散到一个数组中，不同线程会命中到数组的不同槽中，各个线程只对自己槽中的那个值进行CAS操作，这样热点就被分散了，冲突的概率就小很多。如果要获取真正的long值，只要将各个槽中的变量值累加返回。 





# ConcurrentHashMap扩容



<embed src="java8_ConcurrentHashMap扩容.svg" width="100%" type="image/svg+xml"/>
## java7

segment数组的大小在初始化后就不会改变，默认为16。只有segment里面的table可以扩容，table扩容时，会获取ReentrantLock，后续的写操作会阻塞。

接下来的扩容与HashMap类似。



## java8

扩容时，会新建一个容量为原来2倍的nextTable数组，这个数组是多个线程共享的。

1. A、B线程同时对table[0]进行转移，由于synchronized锁的保障，A线程在转移的同时，B线程会被synchronized锁阻塞 。
2. table[0]转移结束后，原来table[0]的地方会变为ForwardingNode(fwd，转发节点)，fwd用于表明该节点已处理完毕。对于读操作，fwd还能指引到新数组进行读操作。
3. A线程开始处理table[1]。
4. A线程处理完毕，将table[1]改为fwd。
5. B线程准备处理table[1]，发现该节点为fwd，表明已经被处理过，跳过该节点。
6. A线程处理table[2]，发现是null，于是可以直接将fwd放入table[2]。
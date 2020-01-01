---
title: Guava限流
categories:
  - 微服务
tags:
  - Guava
  - 限流
date: 2019-10-31 16:48:03
---

# Guava限流

在Guava中提供的限流算法是令牌桶算法。默认是SmoothBursty模式。

<!-- more --> 

Guava实现与理论会有点不同。从理论上来看，令牌会定时地被加入到桶中，而在Guava的实现中，并没有定时器在运行，因为定时器这种实现方式太浪费资源。Guava使用的是基于时间差的令牌延时生成算法，具体来说，有以下两点：

1. **延时**：只有到需要使用限流器时，才去计算。如果新建了限流器，但一直没使用，这个限流器是不会去生成令牌的。
2. **时间差**：需要生成令牌时，会拿当前时间与最后一次生成的时间做一个是时间差，然后根据生成速率可以计算出这段时间应该有多少个令牌生成。

另外，Guava提供的限流器是不保证公平的。因为Guava使用synchronized来锁操作，而synchronized锁是非公平的。

创建限流器的时候会启动一个计时器，记录下创建时的时间。

```java
//限流器获取令牌
public double acquire(int permits) {
    //将令牌数转换为需要等待的时间
    //返回0代表不需要等待
    long microsToWait = reserve(permits);
    
    //如果需要等待，会调用Thread.sleep(time)进行休眠
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
}


//限流器核心流程
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    //生产令牌
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    
    //计算本次需消耗多少令牌
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    
    //计算本次需等待多少令牌
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros = storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
        + (long) (freshPermits * stableIntervalMicros);

    try {
        this.nextFreeTicketMicros = LongMath.checkedAdd(nextFreeTicketMicros, waitMicros);
    } catch (ArithmeticException e) {
        this.nextFreeTicketMicros = Long.MAX_VALUE;
    }
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
}


//令牌生成
void resync(long nowMicros) {
    if (nowMicros > nextFreeTicketMicros) {
        //生产令牌，用时间差除以生产速率，可以得到现在应该有多少令牌
        storedPermits = min(maxPermits,
                            storedPermits
                            + (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros());
        nextFreeTicketMicros = nowMicros;
    }
}
```


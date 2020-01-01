---
title: 基于Redis的分布式锁
categories:
  - Redis
tags:
  - 分布式锁
  - ZooKeeper
  - MySQL
date: 2019-12-30 16:22:48
---

# 基于Redis的分布式锁

<!-- more --> 

互斥性

防止未执行完，锁就过期（后台启动续命线程）



阻塞和非阻塞

未获取到锁的线程可阻塞（CAS），可非阻塞。



可重入性

加锁之前先查看本线程的锁ID是否已上锁



安全性

防止死锁（1.设置过期时间；2.占用锁与设置过期时间应该为原子操作）

防止锁被其他线程任意释放（使用UUID+线程ID作为锁ID）



高可用

防止Redis单机故障（1.Redis使用集群方案，进行数据分片；2.每次取回多个锁）



Redisson分布式锁使用hash作为底层数据结构，hash的key为lockName，field为UUID+threadId。

为了防止锁被其他线程随意释放，field应该对其他线程保密。

获取锁的线程会启动一个后台线程，该线程用于延长锁的过期时间，每隔10秒重新设置30秒的过期时间。



获取锁

```Lua
if (redis.call('exists', name) == 0) then --判断是否存在锁
	redis.call('hset', lockName, UUID+threadId, 1);
	redis.call('pexpire', lockName, expireTime);
	return nil;
end;
if (redis.call('hexists', lockName, UUID+threadId) == 1) then --判断该锁是否是自己线程的锁
	redis.call('hincrby', lockName, UUID+threadId, 1);
	redis.call('pexpire', lockName, expireTime);
	return nil;
end;
return redis.call('pttl', lockName); --获取锁失败，返回锁的过期时间
```





释放锁

```Lua
if (redis.call('hexists', lockName, UUID+threadId) == 0) then --无锁则不需要释放
	return nil;
end;
local counter = redis.call('hincrby', lockName, UUID+threadId, -1); --释放锁
if (counter > 0) then
	redis.call('pexpire', lockName, expireTime); --因锁的可重入性，还未完全释放
	return 0;
else
	redis.call('del', lockName); --完全释放锁
	redis.call('publish', channelName, message);
	return 1;
end;
return nil;
```





方案缺陷

在CAP理论中，Redis属于AP架构。主从架构下，可能会导致多个客户端拿到锁，失去互斥性。

一个客户端从master拿到了锁，在主从同步完成之前，master宕机，slave晋升为master。由于slave没有锁信息，这时第二个客户端可以进行加锁。





针对这个提问，Redis的作者提出了RedLock。RedLock的思想是使用奇数个Redis master，这些master完全独立，没有哨兵、主从、集群之类的关系。从其中的一个master上获取锁与上述方案一致，只要拿到半数以上的锁，视为获取分布式锁成功。如果其中一台master发生故障，切换为无锁信息的slave，也不会失去互斥性。如果如果同时宕机的数量超过半数以上，那还是会丢失互斥性，只是这种事情发生的概率较低。





分布式锁其他方案

Zookeeper属于CP架构

mysql锁行
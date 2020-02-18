---
title: Mybatis数据源与连接池
categories:
  - Mybatis
tags:
  - 池化技术
date: 2020-01-11 15:46:18
---



对于ORM（Object Relational Mapping）框架而言，数据源的组织是一个非常重要的一部分，这直接影响到框架的性能问题。本文将通过对MyBatis框架的数据源结构进行详尽的分析，并且深入解析MyBatis的连接池。

本文首先会讲述MyBatis的数据源的分类，然后会介绍数据源是如何加载和使用的。紧接着将分类介绍UNPOOLED、POOLED和JNDI类型的数据源组织；期间我们会重点讲解POOLED类型的数据源和其实现的连接池原理。





图片： {% asset_img name.png This is an example image %} 

svg：<embed src="AQS_queue.svg" width="100%" type="image/svg+xml"/>

 摘要：<!-- more --> 


MyBatis将数据源分为三种类型：
1. unpooled，不使用连接池的数据源
2. pooled，使用连接池的数据源
3. JNDI，使用JNDI实现的数据源
相关的类分别在以下路径中：
org.apache.ibatis.datasource.unpooled
org.apache.ibatis.datasource.pooled
org.apache.ibatis.datasource.jndi



# 不使用连接池

每次获取连接的时候都需要新建。


```java
//unpooled数据源
  private Connection doGetConnection(Properties properties) throws SQLException {
    initializeDriver();
    //通过drive创建新的连接
    Connection connection = DriverManager.getConnection(url, properties);
    configureConnection(connection);
    return connection;
  }
```



# 使用连接池

对于pooled数据源，它有两个队列：空闲队列和活跃队列。

空闲队列中保存的是空闲的连接，活跃队列中保存的是正在被使用的连接。

当我们需要获取连接时，会按照以下步骤获取：

1. 从空闲队列中获取连接；
2. 空闲队列为空，则去查看活跃队列：
   21. 活跃队列未满，则新建连接并放入活跃队列；
   
   22. 活跃队列满，则挑选最早的连接查看是否过期：
   
       221. 最早的连接已过期，替换掉该连接；
   
       222. 最早的连接未过期，线程阻塞，等待唤醒。

```java
//pooled数据源，代码经过简化
private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    while (conn == null) {
      //从连接池中拿连接是互斥的
      synchronized (state) {
        if (!state.idleConnections.isEmpty()) {
          //1. 若空闲队列中有连接，则直接使用该连接
          conn = state.idleConnections.remove(0);
        } else {
          if (state.activeConnections.size() < poolMaximumActiveConnections) {
            //2.1 活跃队列未满，则新建连接并放入活跃队列
            conn = new PooledConnection(dataSource.getConnection(), this);
          } else {
            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
            if (longestCheckoutTime > poolMaximumCheckoutTime) {
              //2.2.1 活跃队列已满，最早的连接已过期
              //善后最早的连接并替换掉该连接
              state.claimedOverdueConnectionCount++;
              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
              state.accumulatedCheckoutTime += longestCheckoutTime;
              state.activeConnections.remove(oldestActiveConnection);
              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                try {
                  oldestActiveConnection.getRealConnection().rollback();
                } catch (SQLException e) {
                
                }
              }
              conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
              conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
              conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
              oldestActiveConnection.invalidate();
            } else {
              //2.2.2 活跃队列已满，最早的连接未过期
              //线程进入阻塞等待唤醒
              try {
                if (!countedWait) {
                  state.hadToWaitCount++;
                  countedWait = true;
                }
                long wt = System.currentTimeMillis();
                state.wait(poolTimeToWait);
                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
              } catch (InterruptedException e) {
                break;
              }
            }
          }
        }
      }
    }

    //省略
    return conn;
}


protected void pushConnection(PooledConnection conn) throws SQLException {
    synchronized (state) {
        state.activeConnections.remove(conn);
        if (conn.isValid()) {
            if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
                state.idleConnections.add(newConn);
                conn.invalidate();
                state.notifyAll();
            } else {
                conn.getRealConnection().close();
                conn.invalidate();
            }
        }
    }
}
```


---
title: ThreadPoolExecutor
categories:
  - Java知识
tags:
  - 并发编程
  - JDK源码
date: 2019-09-20 09:54:47
---

# ThreadPoolExecutor

<!-- more --> 

线程池共有7个参数可设置

- **corePoolSize** 线程池核心线程数
- **maximumPoolSize** 线程池最大线程池
- **keepAliveTime** 空闲线程存活时间
- **unit** 存活时间单位
- **workQueue** 工作队列
- **threadFactory** 创建线程的工厂
- **handler** 拒绝策略

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
//...
}
```



<embed src="ThreadPoolExecutor.svg" width="100%" type="image/svg+xml"/>


当线程数量大于core时，满足以下两个条件，线程池会开始回收线程：

1. 任务队列为空；
2. 线程等待任务超过一定时间。



ThreadPoolExecutor

```java
//ThreadPoolExecutor.java
//execute的参数是Runnable，如果准备执行Callable，Callable会被包装成Runnable再调用该方法
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();

    //线程池中的线程数量小于corePoolSize，此时新增线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    //线程数量已达到corePoolSize，此时将任务放入队列，不新增线程
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }

    //队列已满，此时再次新增线程
    else if (!addWorker(command, false))
        reject(command);
}
```





Worker

```java
//ThreadPoolExecutor.java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //对于接收Runnable任务的线程池而言，这里的task.run就是运行任务本身
                    //对于Callable而言，run的是Futuretask。Futuretask将Callable包装成了Runnable
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```



```java
//FutureTask.java
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                //这里包装了用户的Callable任务
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```



```java
//FutureTask.java
private void finishCompletion() {
    //waiters是一个线程安全的栈，里面保存了正在等待future结果的线程
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    //唤醒在future.get等待着的线程
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
    done();
    callable = null;        // to reduce footprint
}

```

```java
//FutureTask.java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            //若任务还在执行中，则将该线程作为等待节点压入栈中
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
            //阻塞线程，等待FutureTask完成
            LockSupport.park(this);
    }
}
```

在FutureTask的set(result)中，除了改变任务的状态外，还唤醒了在等待的节点。
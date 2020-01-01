---
title: Reference
categories:
  - Java知识
tags:
  - JDK源码
date: 2019-12-06 11:17:42
---

# Reference

Reference类有个静态的代码块，在这里面会运行一个后台线程，用于将被回收的对象放入引用队列。将对象放入引用队列便于后续进一步的清理操作。

<!-- more --> 

```java
private static class ReferenceHandler extends Thread {
    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }
    public void run() {
        for (;;) {

            Reference r;
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    Reference rn = r.next;
                    pending = (rn == r) ? null : rn;
                    r.next = r;
                } else {
                    try {
                        lock.wait();
                    } catch (InterruptedException x) { }
                    continue;
                }
            }

            // Fast path for cleaners
            if (r instanceof Cleaner) {
                ((Cleaner)r).clean();
                continue;
            }

            //把将要回收的对象放入引用队列
            ReferenceQueue q = r.queue;
            if (q != ReferenceQueue.NULL) q.enqueue(r);
        }
    }
}

static {
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
         tgn != null;
         tg = tgn, tgn = tg.getParent());
    Thread handler = new ReferenceHandler(tg, "Reference Handler");
    handler.setPriority(Thread.MAX_PRIORITY);
    handler.setDaemon(true);
    handler.start();
}
```


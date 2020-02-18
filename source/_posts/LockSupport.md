---
title: LockSupport
categories:
  - Java知识
tags:
  - 并发编程
  - JDK源码
date: 2019-10-15 00:04:49
---

# LockSupport

<!-- more --> 

LockSupport类是基本的线程阻塞组件，在锁和其他同步器中都会使用到。park和unpark是它最核心的两个方法。

LockSupport替代了Thread.suspend和Thread.resume的功能，Thread的suspend和resume方法已经在JDK2中被弃用，原因是suspend方法在挂起线程时，并不会释放该线程持有的锁，这样可能会导致死锁。

```java
//LockSupport.java
//挂起
public static void park() {
    UNSAFE.park(false, 0L);
}

//唤醒
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

如果已经存在一个许可的话（这里的许可可以通过调用unpark获取），调用park会立即返回。

如果没有许可的话，调用park会阻塞，需等待其他地方调用unpark唤醒。

许可不能累加，也就是说如果多次调用unpark，许可还是只有一个，只能允许后面一个park调用立即返回，第二个park会阻塞。

park方法应该放在一个循环中使用，因为会存在虚假唤醒的情况。

在LockSupport的实现中，UNSAFE.park()和UNSAFE.unpark()是native方法。在openjdk/hotspot/src/share/vm/prims/unsafe.cpp可以找到UNSAFE的实现（`thread->parker()->park(isAbsolute != 0, time);`）UNSAFE调用了Parker类的方法。以下为linux下park、unpark的实现。有关许可的部分，是借助了_counter变量来实现的。

该实现主要使用了Pthread中的互斥锁和条件变量。有关Pthread可参考。

```c++
//openjdk/hotspot/src/os/linux/vm/os_linux.cpp
//已删去本次不关注的部分
void Parker::park(bool isAbsolute, jlong time) {
  //检查是否调用过unpark唤醒，若是，则不阻塞
  if (Atomic::xchg(0, &_counter) > 0) return;

  //条件等待时，需要加锁
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }

  //再次检查是否调用过unpark
  int status ;
  if (_counter > 0)  {
    //之前调用过unpark，存在一个许可，可立即返回
    _counter = 0;
    status = pthread_mutex_unlock(_mutex);
    return;
  }

  //进行条件等待
  if (time == 0) {
    status = pthread_cond_wait (&_cond[_cur_index], _mutex) ;
  } else {
    status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime) ;
  }

  //等待到某个条件，释放锁
  _counter = 0 ;
  status = pthread_mutex_unlock(_mutex) ;

}


void Parker::unpark() {
  //加锁
  status = pthread_mutex_lock(_mutex);
  s = _counter;
  //将_counter设置为1，即使unpark在park之前被调用，park阻塞的线程也能被唤醒
  _counter = 1;
  if (s < 1) {
    //如果s<1，说明有线程阻塞，需要发信号唤醒
    if (_cur_index != -1) {
      if (WorkAroundNPTLTimedWaitHang) {
        status = pthread_cond_signal (&_cond[_cur_index]);
        status = pthread_mutex_unlock(_mutex);
      } else {
        status = pthread_mutex_unlock(_mutex);
        status = pthread_cond_signal (&_cond[_cur_index]);
      }
    } else {
      pthread_mutex_unlock(_mutex);
    }
  
  //s>=1，说明当前无线程阻塞
  } else {
    pthread_mutex_unlock(_mutex);
  }
}
```

```c++
class Parker : public os::PlatformParker {  
private:  
  volatile int _counter ;  
  ...  
public:  
  void park(bool isAbsolute, jlong time);  
  void unpark();  
  ...  
}  
class PlatformParker : public CHeapObj<mtInternal> {  
  protected:  
    pthread_mutex_t _mutex [1] ;  
    pthread_cond_t  _cond  [1] ;  
    ...  
}
```

每个Java线程持有一个Parker实例，每个Parker实例中持有一个条件变量和互斥锁，因此当调用unpark唤醒的时候，是可以指定唤醒哪个线程的。






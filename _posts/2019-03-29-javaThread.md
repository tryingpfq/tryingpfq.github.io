---
layout:     post
title:      java线程
subtitle:   有关java线程中线程的生命周期和局部变量的线程安全问题。
date:       2019-03-29
author:     tryingpfq
header-img: img/post-bg-gc1.jpg
catalog: true
tags:
    - 并发编程
---

> 接下来主要是聊聊，有关java线程的生命周期，和java线程几种状态是如何转换的，一起有关线程中局部变量
的一些线程安全问题。

### 通用的线程生命周期
   通用的线程生命周期基本就五种状态了，如下图所示：
   ![通用线程状态](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-th1.png?raw=true)
   <p align = "center">通用线程状态</p>
   * 1：**初始状态**,指的是线程已经被创建，这里仅仅是编程语言层面的创建，但对于操作系统来说，还不运行分配cpu执行。
   * 2：**可运行状态**，指的是线程可以分配CPU执行，在这种状态下，对于操作系统来说，是真正的创建了线程。
   * 3：**运行状态**，指的是线程占有了CPU资源。
   * 4：**休眠状态**，就是当运行状态的时候，线程如果调用一个阻塞API，如IO，或者等待某个事件，就会变为休眠状态。
   * 5：**终止状态**，线程执行完成，或者异常终止。
这里针对的是所有线程状态，不区分编程语音。

### Java线程的生命周期
首先要知道，java线程是有六种状态，分别是New(初始状态)、RUNNABLE(运行/可运行状态)、BLOCKED(阻塞状态)、WATITING(无限等待状态)、
TIMED_WAITING(有限等待状态)、TERMINATED(终止状态)。

但在操作系统中，java线程的BLOCKED、WATITING、TIMED_WAITING是一种状态，即休眠状态，也就是说，如果线程处于这三种状态之一，
那么这个线程就永远没有CPU的使用权。下面看下先看下线程状态的转换图。

  * 1：RUNNABLE与BLOCKED的转换。
     * ：只有一种场景会触发这种转换，就是线程等待synchronized的隐式锁。synchronized 修饰的方法、代码块同一时刻只允许一个
     线程执行，其他线程只能等待。在这种情况下，等待的线程就会从RUNNABLE转换到BLOCKED状态。而当等待线程获得锁后，就又会从
     BOLCKED状态转为RUNNABLE状态。
     * ：平时我们所谓的java在调用阻塞式API时，线程会阻塞，指的是操作系统线程的状态，并不是java线程的状态。
  * 2：RUNNABLE与WAITING的状态转换。
    * ：获得synchronized隐式锁，调用无参的Object.wait()方法。
    * ：调用无参数的Thread.join()方法。其中jion()是一种线程同步方法。例如线程A，当A.join()方法时，当执行这条语句线程B，
    会等待A线程执行完返回后，才可以继续往下执行。那么B这个时候就会从RUNNABLE->WAITING状态转换。当A执行完后，又会从
    WAITING->RUNNABLE状态转换。
    * ：调用 LockSupport.park() 方法。其中的 LockSupport 对象，也许你有点陌生，其实都是基于它实现的。调用LockSupport.part()
    方法，当前线程会阻塞，RUNNABLE->WAITING状态转换，当再调用LockSupport.unpart(Thread th)可唤醒目标线程。WAITING->RUNNABLE状态转换。
  * 3：RUNNABLE与TIMED_WAITING的状态转换(与WAITING区别就是加了超市参数)
    * ：调用带超时参数的 Thread.sleep(long mil)方法。
    * ：获得 synchronized 隐式锁的线程，调用带超时参数Object.wait(long timeout)方法。
    * ：调用带超时参数的 Thread.join(long millis)方法。
    * ：调用带超时参数的 LockSupport.parkNanos(Object blocker, long deadline)方法。
    * ：调用带超时参数的 LockSupport.parkUntil(long deadline)方法。
  * 4：从NEW到RUNNABLE
    * ：Java刚创建出来的线程就是NEW状态，其中主要有两种方式，继承Thread，重写run方法，实现Runnable接口，重写run方法。
    * ：New状态的线程，是不会被操作系统调度的，也就是不会执行，只用转换到RUNNABLE状态，那么就需要调用start()方法。
  * 5：从RUNNABLE到TERMINATED状态
    * ：线程正常运行完成，就会自动终止。如果run方法中，跑出异常，没有捕获，也会导致线程终止。
    * ：强制终止线程stop()方法，当然这个方法是不推荐用的。正确的姿势是调用interrupt()方法。
    
   
---
layout:     post
title:      java线程
subtitle:   有关java线程中线程的生命周期和局部变量的线程安全问题。
date:       2019-03-29
author:     tryingpfq
header-img: img/post-bg-th.jpg
catalog: true
tags:
    - 并发编程
---

> 接下来主要是聊聊，有关java线程的生命周期，和java线程几种状态是如何转换的，以及有关线程中局部变量
的一些线程安全问题。

### 通用的线程生命周期
   通用的线程生命周期基本就五种状态了，如下图所示：
   ![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-th1.jpg)
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
![java线程状态转换图](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-th2.jpg)
 <p align = "center">Java线程状态转换图</p>
 
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
    
### 创建多少才算是合适呢？
   使用多线程就是为了提高性能，但如果创建的线程不合适，自然也会影响一定的性能，性能的指标主要是吞吐量和延迟，所以要提高系统性能，
那么主要是减少延迟和提高吞吐量。创建多少线程要针对系统是CPU密集还是IO密集。

#### CPU密集
CPU密集，说明cpu资源利用比较充分，浪费少，所以这个时候可以尽量减少CPU切换，理论这，线程的数量 = CPU核数，不过
在工程上，线程数量会 = CPU核数 + 1。

#### IO密集
   对于I/O密集的场景，一般会有一个公式进行计算。
   
    最佳线程数 = （I/O耗时 / CPU 耗时） + 1
    
### 局部变量为什么是线程安全的
   有没有想过呢，我们在写代码的时候，如果在方法中定义的局部变量，我们是不用考虑线程安全问题的，也就是不会存在竞争。
   

#### 方法是如何执行的
 其实在方法的调用过程中，会有方法入口地址，每条语句也会有地址的，那问题是怎么找到这些地址依次进行执行呢。
 这个时候就要用到栈，因为栈是先后出的。CPU会有个堆栈寄存器，假设方法A 调用的方法B B调用C，在运行的时候，
 会构建**调用栈**，然后每个方法在调用栈中，会有自己独立的空间，也就是**栈帧**，想必在JVM中对这个会有一定的了解。
 然后栈帧中存放的是什么呢，栈帧存放的是对应方法的参数和返回地址。当调用方法时，会创建新的栈帧，并压入调用栈；当方法
 返回时，对应的栈帧就会被弹出，这就是这个调用的方法执行完成，可见栈帧和方法是同存亡的。
 
#### 局部变量
上面说了方法参数和方法执行完后的返回地址是存放在栈帧中，那么方法中的局部变量是存放在哪里呢？在JVM中，我们可能都知道，
new 出来的对象会存放在堆中，局部变量会存放在虚拟机栈中，所以自然局部变量是存放在该方法对应的栈帧当中。因为虚拟机栈就是
线程私有的，可见局部变量也是和方法共存亡的。

### 调用栈和线程关系
其实上面已经提到过，方法是线程私有的，每个线程都有自己独立的调用栈，每调用一个方法，就会在调用栈中创建对应方法的栈帧，栈帧当中
又存放着方法参数、局部变量、返回地址。所以每个线程都会有自己独立的调用栈，线程与线程之间是不会存在影响的。

**有没有想过为什么递归调用太深，可能导致栈溢出？**

参考[极客时间-Java并发编程]()
   
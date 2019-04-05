---
layout:     post
title:      其他问题
subtitle:   日常中其他问题和知识点记录。
date:       2019-03-08
author:     tryingpfq
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 其他问题
---

> 记录其他知识点，或者日常记录一些碎片化知识点。

* [基础知识](#基础知识)

* [集合](#集合)

* [并发](#并发)

* [JVM](#Jvm)

* [网络知识](#网络知识)

* [算法](#算法)

* [数据库](#数据库)


### 基础知识


### 集合

### 并发
   * 1：long型变量在32位机上读写会出现并发问题吗？
   
       这其实是一个原子问题，因为long型是64位，如果是32位机的话，则在cpu上的写操作要分成2次，一次写高32位，
       另一次写低32位。在单核CPU的场景下，同一时刻只有一个线程执行，会禁止CPU中断，也就是禁止了线程的切换，所以这两次
       写是具有原子性的，不会出现问题。但在多核场景下，同一时刻，可能有多个线程在执行，如果两个线程同时在写变量的高32位，
       就会出现诡异的问题了。
       
   * 2：若有一个共享变量abc，在一个线程里设置了abc = 3,有哪些办法可以让其他线程能够看到abc = 3？
   这个问题可以从java内存模型规范中去解答。
        * 1：可以通过volatile保证共享变量的可见性。
        * 2：使用原则变量。
        * 3：对abc写的时候，加锁。
        * 4：private修饰，通过set()方法修改，并通过get()方法获得。(这个方法保留)
        
   * 3：看一下下面这段代码能解决并发问题吗？
   ```java
        class SafeCalc {
          long value = 0L;
          long get() {
            synchronized (new Object()) {
              return value;
            }
          }
          void addOne() {
            synchronized (new Object()) {
              value += 1;
            }
          }
        }
 ```
   很明显肯定是不可以的，两次都new了，锁的不是同一对象。加锁本质就是在要锁的对象的对象头中写入当前线程ID，new Object
   每次在内存中都是新的对象，所以加锁无效。
   其实经过JVM逃逸分析，这个sync代码会直接被优化掉。运行时的代码是无锁的。
   
   * 4：看下这段代码，如果锁住的不是lalLock对象，而是this.balance,这样可行吗？
     ```java
        class Account {
          // 锁：保护账户余额
          private final Object balLock = new Object();
          // 账户余额  
          private Integer balance;
         
          // 取款
          void withdraw(Integer amt) {
            synchronized(balLock) {
              if (this.balance > amt){
                this.balance -= amt;
              }
            }
          } 
          // 查看余额
          Integer getBalance() {
            synchronized(balLock) {
              return balance;
            }
          }
        }
     ```
   其实我开始以为是可行的，先看看自己错误理解呀，以为锁住字段就可以了，而且之前还不知道，锁的是对象，假如int就不行了。
   为什么锁balock是可以了，因为是一个不可变对象，多个线程拿到balock是一样的，故这样加锁是可以的。如果锁住的是this.balance是不可的，
   这是一个可变对象，当有线程对其修改后，这个字段就换成了一个先的对象（注意，这个Account对象是没改变），此时就出现了多个锁，锁同一资源。
   注意，锁对象是不能用可变对象的。当然问题中，说Integer -128到127是缓存的，如果在这个区间段，是不是可以锁呢，你觉得呢，肯定不可以呀，你想
   一下，在这个区间段有多少代码在使用，这不要卡死吗。
   
   * 5：开启10000个线程，每个线程给员工表的money字段【初始值是0】加1，没有使用悲观锁和乐观锁，但是在业务层方法上加了synchronized关键字，
   问题是代码执行完毕后数据库中的money 字段不是10000，而是小于10000 问题出在哪里？
   这个问题的解答可以参考[开源中国](https://my.oschina.net/u/3777556/blog/3011167)
   
   * 6：线程中断。(当某个线程被其他线程中断后，其他线程在睡眠或者阻塞的时候可以检测到异常)。看下面这段代码会有睡眠问题
```java
    Thread th = Thread.currentThread();
    while(true) {
      if(th.isInterrupted()) {
        break;
      }
      // 省略业务代码无数
      try {
        Thread.sleep(100);
      }catch (InterruptedException e){
        e.printStackTrace();
      }
    }
```
本意是通过th.isInterrupted()来检测线程是否被中断，如果中断了就退出循环。当其他线程通过调用th.interrupt()来中断th线程时，
会设置th线程的中断标志位，从而th.isInterrupted()返回true，退出while循环。

这看上去是没什么问题，实际上几乎起不了作用，这段代码大部分时间都是阻塞在sleep上，当其他线程通过调用th.interrupt()来中断th
线程时，大概率会触发InterruptedException异常，**在触发InterruptedException异常的同时，JVM会把线程的中断标志位清除，所以这个
时候th.isInterrupted()返回false。看正确的代码：
```java
try {
  Thread.sleep(100);
}catch(InterruptedException e){
  // 重新设置中断标志位
  th.interrupt();
}

```
   
### Jvm

### 算法
   * 1：今天突然想到一个关于数组的问题，就是为什么数据初始位置从0开始？
        
        首先，可能是历史原因问题，因为C语言一开始就是从零开始，后来一些高级语言，考虑学习成本，就沿用了。
        
        但从分析上来说可能是另一个原因：
        >假设数据int[] a,那么a是数组的首位置，a[0]也就是偏移量为0的地址，也就是首地址，如果要
        查找a[k]，即编译量为k的地址的值,根据公式：addr = base_addr(a) + k * type_size(int 4字节)。

### 网络知识
   * 1：为什么有了IP地址还需要MAC地址，有了唯一的MAC地址还需要IP地址？
  
      因为对于用户所在的终端（计算机），分配的IP地址是会随时变化的，而且对于各个局域网中，IP可能是相同的。
      为何不可以通过唯一的MAC地址来找到目标终端呢，MAC地址是在网卡中的，由厂商规定。可以举个身份证的例子来
      解释这个问题，身份证号是可以唯一标识一个人的，但通过身份证号是不能找到这个人在何处，首先，前六位是可以确定
      县地址，但知道县也不能找到这个人，应为现居地和出身地不一定相同。同样MAC地址中，有标识知道厂商。所以网络中需要
      MAC地址和IP地址，MAC地址好比身份证号，IP地址好比现居地。如果真的用MAC地址来解决这个问题，那得要一个多大的MAC
      地址表，而且MAC地址是同ARP广播寻找，那可想想网络中的ARP广播风暴。IPv6是可以解决这种问题的，但目前协议还是需要
      MAC的。
      
      记住IP地址是有定位功能，MAC地址是有唯一标识功能，所以两个必须一起合作寻找才能完成。当然，如果是在一个私网当中，
      是可以直接用MAC地址传输的。
      
      [参考-趣谈网络协议-第一讲]()
   * 2：怎么查找IP地址呢？
   
   相必都知道在windows中，可以用ipconfig,在linux 中可以用ifconfig ip addr(这两个命令有什么区别呢)，IP地址是如何
   来的呢 [参考-趣谈网络协议-第3讲]()

### 数据库

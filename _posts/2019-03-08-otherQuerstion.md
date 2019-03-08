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
   
### Jvm

### 网络知识

### 数据库

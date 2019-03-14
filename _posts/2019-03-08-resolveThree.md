---
layout:     post
title:      如何解決并行编程源头问题
subtitle:   可见性、原子性、有序性所带来的问题如何解决
date:       2019-03-08
author:     tryingpfq
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - 并发编程
---

>可见性、原子性、有序性导致的问题导致的问题常常让我们难以发现和察觉。 
>本文主要分析可见性、原子性和有序性带来的问题如何解决。

### 如何解决可见性和有序性导致的问题

#### Java内存模型
Java内存模型底层是怎么实现的，主要是通过**内存屏障**（注意着只是一个抽象的概念）禁止重排序
及时编译器根据具体的底层体系架构，将这些内存屏障替换成具体的CPU指令。
针对可见性和有序性问题，首先有必要了解下：Java内存模型---导致可见性的问题是缓存，导致
有序性问题的原因是编译优化，如果能按照需要禁用缓存以及编译优化，那么问题就解决了。java内存模型
是个很复杂的规范，规范了JVM如何提供按需禁用缓存和编译优化的方法。具体来说，这些方法包括vola、
synchronize、final三个关键字、以及七项Happens-Before规则，接下来我们可以依依分析。

#### volatile
例如这一样代码 private volatile int v = 0，它表达的是：告诉编译器，对这个变量的读写，不能用
CPU缓存，必须从内存中读取或者写入。看下下面这个代码示例。假设线程A执行product()方法，
按照volatile语言，会把变量 v = true 写入内存；假设线程B 执行consume()方法，线程B会从内存中
读取变量V,如果线程B 看到 v == true，那么线程B 看到的变量X是多少呢？答案是42，可以看后面分析。

```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void product() {
    x = 42;
    v = true;
  }
  public void consume() {
    if (v == true) {
      // 这里 x 会是多少呢？
    }
  }
}

```
#### Happens-Before规则
Happens-Before的意思:**前面一个操作的结果对于后续操作来说是可见的**。Happens-Before规则
应该是Java内存模型里重要内容，和程序相关的七项规则如下，都是关于可见性的。

* 1：程序的顺序性规则
这条规则是指在一个线程中，按照程序顺序，前面的操作Happens-Before与后续的任意操作。
注意这里的顺序指的是可以用顺序的方式推演程序的执行，如果不影响结果的话指令还是会重拍的，
此时程序的执行就不一定是完全顺序的，编译器保证结果一定 == 熟悉方式推演的结果。
所以 x = 42 Happens-Before v = true;

* 2：volatile变量规则
这条规则可以这样翻译，指对一个变量的写操作，Happens-Before于后续对这个volatile变量的读操作。

* 3：传递性
  这个比较好理解，比如 A Happens-Before B ,B Happens-Before C ,那么 A Happens-Before C
  所以通过这三个规则，就能解释为什么上面的x指是42。
  * x = 42 Happens-Before 写变量 v = true,规则1
  * 写 v = true Happens Before 读变量 v = true,规则2
  * 根据传递性 x = 42 Happens-Before 读变量 v = true;

* 4：管程中锁的规则
这条规则是指的对一个锁的解锁 Happens-Before 与后续于对这个锁的加锁。管程是个什么东西呢，
我只在操作系统只听过，是一种同步的通用原语，Java中，synchronized就是对管程的实现。
顺便解释性在使用synchronized的时候是怎么加锁和解锁的，其实进入同步代码块会自动加锁和解锁，都是编译器
来帮我们实现的。这个是怎么规范的呢，看段代码吧

```java
synchronized (this) { // 此处自动加锁
  x = 10;
  if (this.x < 12) {
    this.x = 12; 
  }  
} // 此处自动解锁

```
线程A执行完代码后x会变为12，并且会自动释放锁，线程B在获得锁，然后加锁进入同步块的时候，
可以看到线程A对X的写操作，也就是说线程B此时看到的x的值为12，这个牛逼吧。

* 5：线程start()规则
这个其实也很好理解的，假如A是B的主线程，在A中调用了b.start()方法，那么子线程能看到A线程
在调用b.start()方法前的操作。即b.start()方法 happens-Before 与线程B中的任意操作。


* 6：join()规则
这个也是比较好理解的，这是关于线程等待，其实在之前文章中，我们就提到了这个方法的使用。
同样假如A是B的主线程，在A中通过调用b.jion()方法，这个时候A线程会停留在这个地方，不会往下执行，
要等B线程执行退出，显然，如果在线程B中，对某些变量做了操作，对于退出后，后续A线程的执行是可见的。
即线程B中的任意操作Happens-Before 于 b.join()方法操作的返回。

* 7：线程中断规则和对象终结归结。
线程中断规则，指的是线程interrupt()方法的调用先行发生于被中断线程代码检测到中断事件的发生。
这个也不难理解，调用了中断方法，对于如果发生中断了，要做的操作是可见的。
对象终结规则，就是换一个对象的初始化完成先行发生于它的finalize()方法的开始。

#### final
final修饰的变量，是告诉编译器，这个变量生儿不变。

### 如何解决原子性导致的问题
原子性问题主要是因为线程切换导致的，如果能保证某一时刻，只有一个线程在执行---**互斥**，就能保证对
共享变量的修改是互斥的，那么无论是单核CPU还是多核CPU，就都能保证原子性了。如何能保证互斥呢，
我们通用的方法就是互斥锁。

我们把一段需要互斥执行的代码称为临界区，线程在尝试进入临界区之前，必须先要获得该临界区锁，
如果成功，才能进入，此时，我们称这个线程持有锁，离开时，线程要释放锁。

如何利用好锁呢，看个例子，就是转账，假如A B C 三个账户，开始账户都是200元，
假设下次 A 给 B转100 ，B 给 C 转100，理论上要保证A B C 账户为 100 200 300，那在这个过程中
如何去加锁呢？
```java
class Account {
  private int balance;
  // 转账
  synchronized void transfer(
      Account target, int amt){
    if (this.balance > amt) {
      this.balance -= amt;
      target.balance += amt;
    }
  } 
}

```
上面这样加锁真的能解决这种问题吗。当然这种问题肯定是用数据库原子性来做比较靠谱，不过这里先撇开。
这种存在关联关系的多个资源，就有些复杂，我们不能通过锁着某个对象来解决，可以类对象锁。
```java
class Account {
  private int balance;
  // 转账
  void transfer(Account target, int amt){
    synchronized(Account.class) {
      if (this.balance > amt) {
        this.balance -= amt;
        target.balance += amt;
      }
    }
  } 
}

```

这种类对象锁一般是不会用的粒度太多，而且这还会导致死锁问题。这种问题怎么解决呢，看[下一篇](http://tryingpfq.top/2019/03/11/deadlock/)

### 小结
java内存模型中的Happens-Before规则，本质上是为了解决可见性问题。A Happens-Before B,则
意味着A事件对B事件来说是可见的，无论A事件和B事件是否发生在同一个线程里。

互斥锁，是比较常见的，只要有了并发问题，我们首先想到的就是加锁，加锁能够保证临界区代码的互斥性。

参考：[极客时间-令哥专栏](https://time.geekbang.org/column/article/84017)
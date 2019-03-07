---
layout:     post
title:      Java并行编程Bug源头
subtitle:   可见性、原子性、有序性是并行编程问题的来源。
date:       2019-03-07
author:     tryingpfq
header-img: img/post-bg-conQsource.jpg
catalog: true
tags:
    - 并发编程
---

> 本文主要分析可见性、原子性和有序性带来的问题。

### 前言
这些年，计算机硬件（Cpu 内存 IO）在不停的迭代发展。自然会出现一些问题，比如CPU和内存的执行
速度差异，以及内存和IO的速度差异更大。然后我们程序里大部分语句都要访问内存。为了合理利用
CPU的高性能，主要有三个方面

* CPU缓存，可以平衡和内存的速度差异。
* 操作系统增加进程和线程，分时复用CPU，可以均衡CPU/IO设备的速度差异。
* 编译程序优化指令，充分的利用缓存。

当然，在改进的时候，也自然会引起一些其他的问题。

### 缓存导致可见性问题。
在单核的年代，因为就一个cpu,也就一个cpu缓存，自然内存和缓存数据是一致的，及多个线程在同一个cpu
缓存上进行写，对另外线程是可见的。如下图所示，线程A B对共享变量V进行修改，互相之间是可见的，
对于任意线程的访问，都会读取到最新的值,即一个线程对共享变量的修改，其他线程能够立即看到。（单核CPU缓存与内存关系图）
![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/conQsource1.png?raw=true)

现在基本都是多核了，然而cpu都有自己的缓存，这个时候cpu缓存与内存数据就可能存在不一致性。
比如下图，线程A在CPU上对变量V进行了修改，而没有立刻刷新到内存中，显然，这个时候线程A
对变量V的操作对于线程B来说是不可见的。这个问题就是硬件的迭代发展给程序员带来的问题。
多核CPU的缓存与内存关系图：
![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/conQsource2.png?raw=true)


下面我们来通过一段代码来看看多核场景下可见性的问题。
```java
public class MuiltyThread {
    private long count = 0;
    //private AtomicLong count = new AtomicLong(0);
    //private volatile long = 0;
    public static void main(String[] args) {
        try{
            System.out.println(cal());
        }catch (Exception e){
            e.printStackTrace();
        }

    }

    public  static long cal() throws InterruptedException {
        MuiltyThread test = new MuiltyThread();
        Thread t1 = new Thread(()->{
            test.add();
        });
        Thread t2 = new Thread(()->{
            test.add();
        });
        t1.start();
       // t1.join();
        t2.start();
        t1.join();
        t2.join();
       //return test.count.get();
        return test.count;
    }

    public void add(){
        int dx = 0;
        while(dx++ < 10000){
            count ++;
            //count.incrementAndGet();
        }
    }
}

```
add()循环10000次，对于我们期望的结果肯定是20000了。但当你每次运行的时候，得到的结果应该都不是20000，是一个在10000-20000的
随机数。为什么会造成这个原因呢，假如开始t1对count+1操作，此时CPU1的缓存为1，但次数t2去执行，是从内存中去读取，而且
t1对count的操作对于t2来说是不可见得，读到的时间可能是0；因为对于寄存器来说，什么时候把结果刷新到内存是不确定的。
所以在这种情况下得到的结果是10000-20000的随机数，当着这不仅仅是可见性问题，其实还和原子性问题有关。如下图所示：
![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/conQsource3.png?raw=true)

在代码中，其实用我注释的代码可以解决这个问题，对于共享变量，存在线程安全问题的时候，可以通过volatile来保证其可见性，但volatile
不能保证原子性，还有就是可以用原子变量。

###  线程切换带来的原子问题
相对于CPU的计算速度来说，IO操作格外慢，为了充分利用CPU资源，早起操作系统就出现了进程，即各个进程之间通过时间片轮转来获取
cpu使用权，也就是任务在CPU上的切换。
如果一个进程要进行IO操作，测此时可以让出CPU，进入休眠状态，其他进程则可占有CPU，当其他进程IO完成后，会被唤醒。
![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/conQsource4.png?raw=true)


早起的操作系统任务调度是基于进程的，现代操作系统中，进程是资源分配的单位，但线程是调度的基本单位。一个进程可以创建多个线程，
并且这些线程是共享同一个内存空间的，显然线程切换比进程切换成本会低。前面所说的任务切换，其实就是现在的线程切换。


Java并发程序都是基于多线程的，然后我们写的java程序（高级语言），一条语句，可能在底层会对应多条CPU指令才能完成。这主要是为了
方便我们开发，和代码的可读性。比如 count += 1,可能就要对应三条CPU指令。
* 1：把count从内存加载到CPU寄存器。
* 2：在寄存器中通过Add指令进行加1操作
* 3：将结果写入到内存，当然由于CPU缓存，可能写入的是缓存，而不是内存。

下面来分析一下为什么线程切换会导致原子问题呢？

记住操作系统在进行任务切换(线程切换)的时刻，这个时刻可以发生在任意一条**CPU指令**执行完成，是CPU指令哦。
而不是我们在高级语言中，自己写的语句。 对应上面的三条指令来说，我们假设count = 0,如果
线程A在指令1执行完后做线程切换，线程A和线程B按照下图的序列执行，那么发现两个线程都执行了
count+=1的操作，但得到的结果并不是我们期望的2，而是1。先看看非原子操作的执行路径图。
![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/conQsource5.png?raw=true)

我们潜意识可能会这样认为，count+=1这个操作是一个不可分的整体，就像一个原子一样，线程的
切换可以发生在count+=1之前，或者之后。但不会发生在中间，如果知道其CPU指令就不难理解。
**我们把一个或者多个操作在CPU执行过程中不可中断的特性称为原子性。** CPU能保证的原子操作是
CPU指令级别，而不是高级语言的操作符，这里是和我们直觉不同的地方。因此，很多时候，我们需要在高级语言层面
来保证操作的原子性。


### 编译优化带来的有序问题。
除了上面两个会导致诡异Bug的技术外，还有就是有序性。这常常也违背我们的直觉，就是编译器
编译的时候，会对我们写的程序代码就行优化，可能改变语句的先后顺序，并不是完全按照我们
写的语句的顺序执行。比如： a +=1，b+=1，通过编译器优化后，可能会反过来，即 b += 1，a +=1,
即指令进行了重排序，当然在这个例子中，是不会出现问题的，因为不会影响最后的执行结果。
不过有时候，编译器及解释权的优化可能导致一些诡异的Bug，而且意想不到哦。

想必，大家都在对创建单例模式的时候，都曾了解过双重检查创建单例对象。例如下面代码在获取实例getInstance()的方法中，
我们首相判断instance是否为空，若为空，则锁定Singleton.class,并在此判断instance是否为空，如果还为空，则创建Singleton实例，
```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}

```
这种问题是很诡异的，我们首先来分析一下，假设照样有两个线程在调用getInstance()方法来获得instance实例，当他们发现第一个判断instatnce == null的时候，假设
线程A先获取锁，然后线程B就必须等待不能进入同步代码块，然后A再次判断 instance == null ,则new 一个实例，之后释放锁。这个时候线程B就可以获得锁，进入
第二个判断 instance == null ,此时发现不再为null，线程B就不会再创建Singleton实例。这个过程看上去是很完美的，但其实这里还是会存在问题的，因为在new操作上，会分为三步。
* 1：申请分配一块内存M。
* 2：在内存M上初始化初始化Singleton对象。
* 3：然后M的地址复制给instance。

但经过编译器优化后，会发生指令重排序(**可以通过volatile**防止)，实际的执行顺序会是这样的：1 --> 3 --> 2;
在这种情况下会发生什么问题的，假设开始线程A和线程B，都执行到getInstatnce()方法，而线程A进入第一个判断为空，则开始锁住同步代码块，进入第二个判断为空，
然后就执行new操作:1 -- > 3 --> 2 按照这个顺序执行，当执行到3完成后，注意，这个时候并没有初始化实例Singleton,只是对instance进行了逻辑地址赋值，此时正好发生线程
切换，此时因为线程A并没有退出同步代码块，所以不会释放锁的。而此时线程B不能进入同步代码块的，但可以执行到第一个判断，但此时判断instance 并不为null
但此时的instance是没有经过初始化的，如果在这个时候对instance的成员变量进行了访问，那么问题就来了，
空指针异常就不可思议的跑出。下面看下线程A和线程B执行顺序。
![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/conQsource6.png?raw=true)

对于单例模式，写法比较多，常用的可能是用静态内部类，代码如下，也可见[github-DesignPattern](https://github.com/tryingpfq/DesignPattern)
```java
public class Singleton {
      //先声明一个静态内部类
      //private 私有的保证别人不能修改
      //static 保证全局唯一
      private static class LazyHolder{
          //final 为了防止内部错误操作
          private static final Singleton INSTANCE  = new Singleton2();
      }
      //私有构造方法
      private Singleton(){}
      
      //同样提供静态方法获取实例
      //final 确保别人不能修改
      public static final Singleton getInstance(){
          return LazyHolder.INSTANCE;
      }

}

```
还有类似Spring基于Map实现单例的方式
```java
public class Singleton3 {

    private static Map<String,Singleton3> map = new HashMap<>();
    static {
        Singleton3 single = new Singleton3();
        map.put(single.getClass().getName(),single);
    }

    //保护的默认构造方法
    protected Singleton3(){}

    //静态工程方法，返回此类唯一实例
    public static Singleton3 getInstance(String name){
        if(name == null) name = Singleton3.class.getName();

        if(map.get(name) == null){
            try{
                map.put(name,(Singleton3) Class.forName(name).newInstance());
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        return map.get(name);
    }

}
```



### 小结
在并发编程中，可见性、原子性、有序性的导致Bug出现的源头。并且上文也说了，缓存导致可见性问题、
线程切换带来原子性问题、编译优化带来有序性问题。


参考：[极客时间-令哥专栏](https://time.geekbang.org/column/article/83682)
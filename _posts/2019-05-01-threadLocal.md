---
layout:     post
title:      ThreadLocal
subtitle:   ThreadLocal应用和原理
date:       2019-05-01
author:     tryingpfq
header-img: img/post-bg-gc1.jpg
catalog: true
tags:
    - ThreadLocal
---

> ThreadLocal是一个本地线程副本变量工具类，主要用于将私有线程和该线程存放的副本对象做一个映射，这样的话各个线程之间的变量互不干扰，在高并发的场景下，可以实现无状态的调用，特别适用于各个线程依赖不通的变量值完成操作额场景。比如（数据库连接管理，线程回话管理等场景）。

### ThreadLocal的数据结构模型

![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-threadLocal1.png?raw=true)

从图中可以看出，每个ThreadLocal的核心机制。

* 每个Thread的线程内部都有一个Map。
* Map中存储线程本地对象（key）和线程的变量副本（value）。
* Thread内部的Map是由ThreadLocal维护的，由ThreadLocal负责向map中进行读和写。


    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;


### 看下ThreadLocal的核心方法
get()方法：
1.获取当前线程的ThreadLocalMap对象threadLocals
2.从map中获取线程存储的K-V Entry节点。
3.从Entry节点获取存储的Value副本值返回。
4.map为空的话返回初始值null，即线程变量副本为null，在使用时需要注意判断NullPointerException。


    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }



set()方法：

     public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }


显然，在set的时候，如果遇到hash冲突的时候怎么去解决了。其实ThreadLocalMap结构比较简单，没有采用next指针，也就是解决冲突的时候不是链表的方式，而是采用线性探测的方法。也就是说，每次根据key，计算出hashcode值，确定位置，如果已经被占用了，那么久需要用固定算法，以固定步长左右寻找空位置。具体代码实现就不看了。

如果这个地方每个线程只有一个变量，这样的话所有的线程存放到Map中Key是同一个ThreadLocal,就不会存在冲突问题。


remove()方法：

    public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }


### TheadLocalMap
TheadLocalMap的数据结构比较简单，里面就是维护一个Entry，其中这个Entry的key为ThreadLocal,Value值为对应的T。

     static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;
    
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

Entry继承自WeakReference（弱引用，生命周期只能存活到下次GC前），但只有Key是弱引用类型的，Value并非弱引用。所以这个地方可能会引起内存泄漏问题。

### ThreadLocalMap存在的问题
因为Key是弱引用，但发生GC的时候，这个key（ThreadLocal）没有被强引用，会被回收，也就是此时Entry中的key = null,而Value的值是还存在的。如果当前线程没有被回收，假如线程中中的线程，当不断的set（）的时候，Value就可能形成一条没有回收的引用连，引起内存泄漏。

* 1. ThreadLocalMap中的Entry弱引用（key）到的是threadLocal 对象，此处的弱引用意图是，当外界不再持有对threadLocal 对象的强引用的时候，threadLocal 对象可以被GC。此处Entry对 threadLocal 的弱引用不会引起内存泄露。
* 2. Entry除了持有对threadLocal的弱引用之外还持有对value的强引用。先说为什么调用了threadLocal.remove() 就解决了内存泄漏的风险。threadLocal.remove() 这个方法最终会有机会执行到：entry.value = null;entry = null;
这样就解除了entry对value的强引用关系，当外部也没有对value的强引用的时候，value就可以被GC。

我们知道 entry 是被 threadLocalMap 引用的,而 threadLocalMap 是被 thread 引用的。如果一个thread执行完毕，进入 TERMINATED 状态时，作为一种GC Rootterminated 状态的 thread本身就是可以被GC的。那么thread所引用的 threadLocalMap 也就是可以被GC的。

**那么什么情况下 threadLocalMap 不能被回收呢？**

那就是thread并不会进入 terminated 状态的时候。
什么时候不进入 terminated 呢？就是当 thread 配合线程池使用的情况下，thread在运行完毕之后
会被再次放回线程池。
那么如果这个线程永远不被用到，此处的threadLocalMap 包括entry 和 entry引用的value 就不能被回收了。
那么如果这个线程被再次启用，那么threadLocalMap也就不会再重新初始化了。
此处应该考虑另外一个问题，那就是如果再次调用 threadLocal.get() 方法，得到的是上一次set的内容，也就是脏读了。

目前避免ThreadLocalMap内存泄漏的方法可以下面这样处理：

    ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();
    try {
    	threadLocal.set(new Session("val", "val"));
   		 // 其它业务逻辑
	} finally {
  	  threadLocal.remove();
	}



### 比较

ThreadLocal 和 FastThreadLocal，这里就不展开分析了，可以看下FastThreadLocal的源码，如果业务线程用的是FastThreadLocalThread的时候，用这个还是效率比较快的，当然这个也兼容ThreadLocal。

其实实现很简单，对于每个FastThreadLocal都会有一个唯一的index,在每个FastThreadLocalThread线程中，都会有一个InternalThreadLocalMap，map就有个object[]数组来进行缓存，数据角标就是对应的FastThreadLocal的index,在jdk的中，缓存的是以ThreadLocal作为key。可[参考](http://www.jiangxinlingdu.com/interview/2019/07/01/fastthreadlocal.html)




### 总结
每个ThreadLocal只能保存一个变量副本，如果想要上线一个线程能够保存多个副本以上，就需要创建多个ThreadLocal。
ThreadLocal内部的ThreadLocalMap键为弱引用，会有内存泄漏的风险。适用于无状态，副本变量独立后不影响业务逻辑的高并发场景。如果如果业务逻辑强依赖于副本变量，则不适合用ThreadLocal解决，需要另寻解决方案。

[参考-简书](https://www.jianshu.com/p/98b68c97df9b)


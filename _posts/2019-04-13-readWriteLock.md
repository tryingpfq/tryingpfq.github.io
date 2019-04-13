---
layout:     post
title:      Java读写锁
subtitle:   读写锁的实现和应用。
date:       2019-04-13
author:     tryingpfq
header-img: img/post-bg-rw.jpg
catalog: true
tags:
    - 并发编程
---

> 在Java SDK并发包中，其实针对不同场景会有不同的工具类，分场景优化性能，提升易用性。比如一个常见的常见，
就是读多写少，在并发包中，提供了读写锁，本文，我们主要是来讲解一下读写锁。

### 前言
读写锁的应用，其实是比较常见的，比如缓存，就是典型的读多更新少，那么下面会看看缓存中读写锁的应用。针对这种场景，
Java SDK包中提供了读写锁---ReadWriteLock.

### 概念
读写锁：顾名思义，就是两种不同的锁，分为读锁和写锁，可以允许多个线程读，不会互斥，这是JVM控制的，但是这个时候是不运行
写的，即写锁会被阻塞。当有线程获得写锁的时候，其他其他线程读或者写都会被阻塞的。

读写锁会遵循三条基本规则:
 * 1:允许多个线程同时读共享变量。
 * 2:只允许一个线程写共享变量。
 * 3:如果一个线程正在执行写操作，此时禁止读线程读共享变量。
 
 
下面可以看一个简单的代码示例：
```java
public class TestOne {

    public static void main(String[] args) {
        final MyData data = new MyData();
        for(int i = 0; i < 3 ; i ++){
            new Thread(String.valueOf(i)){
                @Override
                public void run() {
                    data.get();
                }
            }.start();
        }

        for(int i = 3; i < 6 ; i ++){
            new Thread(String.valueOf(i)){
                @Override
                public void run() {
                    data.put(new Random().nextInt(1000));
                }
            }.start();
        }
    }


    static class MyData{
        private Object data = null;     //要读写的数据

        private ReadWriteLock lock = new ReentrantReadWriteLock();

        private Lock r = lock.readLock();   // 读锁

        private Lock w = lock.writeLock();     //写锁

        /**
         * get data (read)
         */
        private void get(){
            r.lock();   //其他线程只能读 不能写了
            System.out.println("Thread - "+Thread.currentThread().getName() + " be ready read the data");
            try{
                Thread.sleep(1000L);
                System.out.println("Thread - "+Thread.currentThread().getName() + " finish read the data :" + data);
            }catch (InterruptedException e){
                e.printStackTrace();
            }finally {
                // release
                r.unlock();
            }
        }

        private void put(Object data){
            w.lock();   // 上写锁 其他线程读或者写都不允许
            System.out.println("Thread - "+Thread.currentThread().getName() + " be ready write the data: " + data);
            try{
                Thread.sleep(10000L);
                this.data = data;
                System.out.println("Thread - "+Thread.currentThread().getName() + " finish write the data: "+ data);
            }catch (InterruptedException e){
                e.printStackTrace();
            }finally {
                // release
                w.unlock();
            }
        }
    }
}

Thread - 1 be ready read the data
Thread - 0 be ready read the data
Thread - 1 finish read the data :null
Thread - 0 finish read the data :null
Thread - 4 be ready write the data: 483
Thread - 4 finish write the data: 483
Thread - 5 be ready write the data: 815
Thread - 5 finish write the data: 815
Thread - 2 be ready read the data
Thread - 2 finish read the data :815
Thread - 3 be ready write the data: 306
Thread - 3 finish write the data: 306

```

### 读写操作快速实现一个缓存
下面看下，如何用ReadWriteLock快速实现一个通用的缓存工具类。加入我们要加载的数据是从数据库读取，
所以当我们要获得数据的时候，先从缓存Cache<K,V>中获取，如果获取失败，那么从数据库获取，若获取到了，并加载到
缓存中，下次要获取同一个对象的时候，直接从缓存中获取即可，所以可以减少对数据库的访问操作。除非对数据有更新的时候，
才更新数据库。

下面先看下示例代码：

```java
public class Cache<K,V> {
    final HashMap<K,V> m = new HashMap<K, V>();

    final ReadWriteLock lock = new ReentrantReadWriteLock();

    final Lock rk = lock.readLock();
    final Lock wk = lock.writeLock();

    /**
     * read cache
     * @param key
     * @return
     */
    private V get(K key){
        V v = null;
        rk.lock();
        try{
            v = m.get(key);
        }finally {
            rk.unlock();
        }

        if(v != null){     //在缓存中
            return v;
        }

        // 写锁
        wk.lock();
        try{
            v = m.get(key);
            if( v == null){     //为什么要再次检查了 因为释放了读锁后，可能有其他线程进行了些操作
                // load on DB
                //v = 访问数据库
                if(v == null){  // not on db 没有符合要查找的数据
                    return null;
                }
                //加入缓存
            }
            m.put(key,v);
        }finally {
            wk.unlock();
        }
        return v;
    }
}

```

上面这个示例，是可以优化一下，就是锁的升级，不过要注意下，读锁是不允许升级为写锁的，一定要先释放读锁，
如果为释放读锁，就去获取写锁的话，会导致写锁永久等待。但是允许在释放写锁前，降级为读锁。

### 总结
    本文主要是针对读写锁的应用，和用读写锁实现缓存。
    
[参考-极客时间-《Java并发编程实战》](https://time.geekbang.org/column/article/88909)



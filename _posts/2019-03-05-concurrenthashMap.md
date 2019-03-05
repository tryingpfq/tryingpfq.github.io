---
layout:     post
title:      ConCurrentHashMap
subtitle:   ConCurrentHashMap1.8源码分析
date:       2019-03-05
author:     tryingpfq
header-img: img/post-bg-concuhashmap.jpg
catalog: true
tags:
     - 集合
     - 源码
---

> map在java中是站很重要地位，如果不存在并发的情况下，hashMap就够了
> 但如果存在并发的时候，我们就好考虑线程安全。
> 本文主要有关ConcurrentHashMap1.8源码分析


### 前言
ConcurrentHashMap就是为了解决HashMap线程不安全的问题，直观感觉就是当Thread1从内存读取map，进行修改没完成，还未写入内存的时候，Thread2也从内存读取map，此时出现了线程不安全的问题。
ConcurrentHashMap主要是通过CAS和Synchronize 保证线程安全，CAS保证数组，synchronize保证数组脚标每个位置下的链表。
### 原理和过程
#### 什么时候需要考虑线程安全呢？
  * 初始化数组大小的时候
  * get方法table[i]定位的具体需要拿到内存中的最新值的时候
  * 扩容的时候
  * fh = f.hash 当前数组小标的节点的hash值，为什么判断这个hash值呢
    hash --> key.hashCode等计算出来的
  * else if ((fh = f.hash) == MOVED)
              tab = helpTransfer(tab, f);
     fh == -1 的时候，说明有线程正在进行扩容，先不进行put操作，帮助一起扩容
  
  相比于HashTable,coucurrentMap在效率上是明显更高效的。
  HashTable是线程安全的，但它是在put、get、remove、clear等方法上做了有锁同步机制，
  即加了synchronize关键字，显然，锁力度太大，浪费，影响了性能
  
#### 源码

在Node节点中的 val 和 next变量前加了 volatile关键字，保证数据的可见性，每次读取到的都是主内存中最新的值
```java
final int hash;
final K key;
volatile V val;
volatile Node<K,V> next;
```
**putAll()方法**
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //tabAt()方法，可以与硬件底层打交道，获取内存最新的值，为什么tab数组已经加了volatile，保证了可见性，保证读取内存最新值，为什么在获取tab数组某一个下标值的时候，还需要这样获取内存最新值呢？虽然tab保证了可见性，只能保证数组引用地址是最新的，但不能保证元素是最新的，
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { //i位置为空，直接new一个节点
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {	//同步代码块，锁对象是当前数组下表的节点，也就是该位置的链表的头结点，自然，对该链表修改是线程安全的了，也不会影响别的链表操作
                //锁力度缩小。
                if (tabAt(tab, i) == f) { //双重check,可能get方法的时候，对其进行了修改
                    if (fh >= 0) {	//链表 和hashMap相似
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {	//红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);//转红黑树
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount); //和扩容有关
    return null;
}
```
**数组初始化方法(CAS 保证单线程初始化)**
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)	//表示已经有一个线程在对数组进行初始化 violate SizeCtl
            Thread.yield(); // 让出时间片
        //保证只有一个线程进行修改
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];	//初始化数组
                    table = tab = nt;
                    sc = n - (n >>> 2);    10000 -  100 = 16 -4 = 12
                }
            } finally {
                sizeCtl = sc;	//相当于HashMap中的扩容 阈值
            }
            break;
        }
    }
    return tab;
}
```
**有三个地方会对concurrentHashMap进行扩容**
* 1：addCount()
* 2：helpTransfer()
* 3：转红黑树的时候，此时长度大于8，但如果节点未超过64的时候，会进行预处理，进行扩容，调整Map节点

**addCount（)方法**
```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2)) 	//此时rs肯定是一个很小的负数，然后为什么又加2呢
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

**put的时候，正在扩容，此时，线程要先去帮助完成扩容，显然此时的nextTable != null**
 ```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);//nextTab 不为空
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range 最小为16，也就是线程领取的扩容任务长度区间大小
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;	//数组初始化，32了
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
//通过--i从高到底,末尾开始，遍历领取的任务
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)
//fwd继承了Nodes,并且将hash设置为MOVED，
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed 
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) { //链表的方式进行迁移
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {//红黑树的方式进行迁移
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

### 小结
以上就是针对concurrentHashMap的实现原理，怎样解决线程安全和扩容过程分析。
---
layout:     post
title:      JVM监控及诊断
subtitle:   Java虚拟机中常用到的监控和诊断工具
date:       2019-03-06
author:     tryingpfq
header-img: img/post-bg-monitorTools.jpg
catalog: true
tags:
    - 监控工具
---

> 对JDK中用于监控和诊断工具做一些了解。

### jps
jps是java提供的一个显示当前所有java进程pid命令，适合在linux/unix平台上简单看当前java进程的一些情况。


可以通过jps命令，打印所有正在运行的java进程相关信息。默认情况下，jps的输出信息包括java进程的
进程ID以及main类名。当然也可以通过追加参数。如 -l 将打印模块名以及包名，-v打印传递给java虚拟机
的参数(如-XX:+UnlockExperimentalVMOptions)，-m打印传递给主类的参数。

    116 RemoteMavenServer
    7092
    13304 Launcher
    6440 GameStart
    5980 Jps
    
### jstat
jstat命令是一个比较常用的，可以用来打印目标java进程的性能数据，比如Survivor区 gc时间，进程启动时间等
当然它还包含很多子命令，如-class:用来打印类加载相关数据；-compiler和-printcomilation:用来打印即时编译
的相关数据；-gcXXX:将可以打印垃圾回收的相关数据。
比如下面命令 jstat -gc 6440 1s 5;(gc:用来查看该进程垃圾回收信息，6440表示进程ID，1s 5 表示1秒打印5次)

     S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
    16896.0 14848.0  0.0   13931.7 160768.0 55894.4   51200.0    19764.9   45912.0 44993.6 5760.0 5595.8      9    0.110   2      0.100    0.210
    16896.0 14848.0  0.0   13931.7 160768.0 55894.4   51200.0    19764.9   45912.0 44993.6 5760.0 5595.8      9    0.110   2      0.100    0.210
    16896.0 14848.0  0.0   13931.7 160768.0 55894.4   51200.0    19764.9   45912.0 44993.6 5760.0 5595.8      9    0.110   2      0.100    0.210
    16896.0 14848.0  0.0   13931.7 160768.0 55894.4   51200.0    19764.9   45912.0 44993.6 5760.0 5595.8      9    0.110   2      0.100    0.210
    16896.0 14848.0  0.0   13931.7 160768.0 55894.4   51200.0    19764.9   45912.0 44993.6 5760.0 5595.8      9    0.110   2      0.100    0.210
    
在输出中，前四列分别表示为两个Survivor区的容量（Capacity）和已近使用（Utility）,很容易发现，两个
Survivor区的容量是一样的，而且有一个区使用为0；

如果需要查看某进程堆压力，可以通过GC时间占运行时间比例计算。使用-t命令就能查看这些参数
    
    $ jstat -gc -t 6440
    Timestamp        S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
             2449.0 16896.0 14848.0  0.0   13931.7 160768.0 55894.4   51200.0    19764.9   45912.0 44993.6 5760.0 5595.8      9    0.110   2      0.100    0.210
             
 我们可以通过比较该进程启动时间和GCT表示GC总的时间，或者两次测量的间隔时间和总GC时间增量，来
 得出GC时间占运行时间的比例。
 
 如果该比例超过20%，则说明目前堆压力较大，如果超过90%，则说明机会没有可用空间，随时可能跑出OOM异常。
 
 jstat还可以用来查看内存泄漏，在性能数据中，可以查看OU值(已近占用的老年代)，如果这个值在呈现上涨的趋势，这
 说明无法回收的对象越来越多，因此很可能出现内存泄漏。
 
### jmap
如果想要分析Java虚拟机中的对象信息，则可以使用jmap命令，同样jmap命令也包括多条子命令
* -clstats，该子命令将打印被加载类的信息。
* -finalizerinfo，该子命令打印所有待finalize的对象
* -histo，该子命令将统计各个类的实例数目以及占用内存，并按照内存使用量从多至少的顺序排列。
此外，-histo:live只统计堆中的存活对象。
* -dump，该子命令将导出Java虚拟机快照。同样，-dump:live只保存堆中的存活对象。

看一下-histo子命令的部分输出
       
     num     #instances         #bytes  class name
    ----------------------------------------------
       1:        428329       36994720  [C
       2:         78803       18384656  [B
       3:        258840        6212160  java.lang.String
       4:          8147        6013704  [I
       5:         63373        3041904  com.sun.tools.javac.file.ZipFileIndex$Entry
       6:         50179        1204296  java.lang.StringBuilder
       7:         11134        1080648  [Ljava.lang.Object;
       8:          9019        1005608  java.lang.Class
       9:         29403         940896  java.util.HashMap$Node
      10:          9310         819280  java.lang.reflect.Method
      11:         25039         801248  com.sun.tools.javac.util.SharedNameTable$NameImpl
      12:             6         786528  [Lcom.sun.tools.javac.util.SharedNameTable$NameImpl;
      13:          8863         709040  com.sun.tools.javac.code.Symbol$ClassSymbol
      14:         27803         667272  com.sun.tools.javac.util.List
      15:         37454         599264  com.sun.tools.javac.file.RelativePath$RelativeFile
      
 由于jmap访问所有对象，为了保证在此过程中不被应用线程干扰，jmap需要借助安全点机制，让所有线程
 停留在不改变堆中数据的状态。也就是说由jmap导出的快照必定是安全点位置。
 
 ### jinfo
 如果想要查看目标进程的参数，如传递给java虚拟机的-X（即输出中的jvm_args）、-XX(输出中的VM Flags)
 、以及可在java层面通过System.getProperty获取的-D参数（输出中System Properties）。
 
 下面就是打印的部分信息
    
    $ jinfo 6440
    Attaching to process ID 6440, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 25.181-b13
    Java System Properties:
    
    java.runtime.name = Java(TM) SE Runtime Environment
    java.vm.version = 25.181-b13
    sun.boot.library.path = C:\Program Files\Java\jdk1.8.0_181\jre\bin
    java.vendor.url = http://java.oracle.com/
    java.vm.vendor = Oracle Corporation
    path.separator = ;
    file.encoding.pkg = sun.io
    java.vm.name = Java HotSpot(TM) 64-Bit Server VM
    sun.os.patch.level =
    sun.java.launcher = SUN_STANDARD
    user.script =
    user.country = CN
    user.dir = D:\project\mmorpg
    java.vm.specification.name = Java Virtual Machine Specification
    java.runtime.version = 1.8.0_181-b13
    java.awt.graphicsenv = sun.awt.Win32GraphicsEnvironment
    os.arch = amd64
    java.endorsed.dirs = C:\Program Files\Java\jdk1.8.0_181\jre\lib\endorsed
    line.separator =
    
    java.io.tmpdir = C:\Users\4083\AppData\Local\Temp\
    java.vm.specification.vendor = Oracle Corporation
    user.variant =
    os.name = Windows 10
    sun.jnu.encoding = GBK
    ......

### jstack
jstack可以用来打印目标Java进程中各个线程的栈轨迹，以及这些线程持有的锁。用jstack可以
用来对死锁的检查。

    $jstack 6640
    "nioEventLoopGroup-2-1" #18 prio=10 os_prio=2 tid=0x000000001b9a6800 nid=0x4d0 runnable [0x000000001eb1e000]
       java.lang.Thread.State: RUNNABLE
            at sun.nio.ch.WindowsSelectorImpl$SubSelector.poll0(Native Method)
            at sun.nio.ch.WindowsSelectorImpl$SubSelector.poll(WindowsSelectorImpl.java:296)
            at sun.nio.ch.WindowsSelectorImpl$SubSelector.access$400(WindowsSelectorImpl.java:278)
            at sun.nio.ch.WindowsSelectorImpl.doSelect(WindowsSelectorImpl.java:159)
            at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
            - locked <0x00000000f592db40> (a io.netty.channel.nio.SelectedSelectionKeySet)
            - locked <0x00000000f592ec78> (a java.util.Collections$UnmodifiableSet)
            - locked <0x00000000f592eb68> (a sun.nio.ch.WindowsSelectorImpl)
            at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
            at io.netty.channel.nio.SelectedSelectionKeySetSelector.select(SelectedSelectionKeySetSelector.java:62)
            at io.netty.channel.nio.NioEventLoop.select(NioEventLoop.java:753)
            at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:409)
            at io.netty.util.concurrent.SingleThreadEventExecutor$5.run(SingleThreadEventExecutor.java:886)
            at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
            at java.lang.Thread.run(Thread.java:748)
    
    "ObjectCleanerThread" #17 daemon prio=1 os_prio=-2 tid=0x000000001b9a2000 nid=0x3248 in Object.wait() [0x000000001e3de000]
       java.lang.Thread.State: TIMED_WAITING (on object monitor)
            at java.lang.Object.wait(Native Method)
            at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)
            - locked <0x00000000eeb14028> (a java.lang.ref.ReferenceQueue$Lock)
            at io.netty.util.internal.ObjectCleaner$1.run(ObjectCleaner.java:54)
            at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
            at java.lang.Thread.run(Thread.java:748)
   
    "Druid-ConnectionPool-Create-589016913" #15 daemon prio=5 os_prio=0 tid=0x000000001b9a1800 nid=0x7b8 waiting on condition [0x000000001b44e000]
       java.lang.Thread.State: WAITING (parking)
            at sun.misc.Unsafe.park(Native Method)
            - parking to wait for  <0x00000000c32a3788> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
            at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
            at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
            at com.alibaba.druid.pool.DruidDataSource$CreateConnectionThread.run(DruidDataSource.java:1837)
    
    "Abandoned connection cleanup thread" #14 daemon prio=5 os_prio=0 tid=0x000000001b9a6000 nid=0x1dd8 in Object.wait() [0x000000001ae8e000]
       java.lang.Thread.State: TIMED_WAITING (on object monitor)
            at java.lang.Object.wait(Native Method)
            at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)
            - locked <0x00000000c386ba90> (a java.lang.ref.ReferenceQueue$Lock)
            at com.mysql.cj.jdbc.AbandonedConnectionCleanupThread.run(AbandonedConnectionCleanupThread.java:70)
            at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
            at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
            at java.lang.Thread.run(Thread.java:748)
    
    "Service Thread" #12 daemon prio=9 os_prio=0 tid=0x000000001746c000 nid=0x174c runnable [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE
    
    "C1 CompilerThread2" #11 daemon prio=9 os_prio=2 tid=0x00000000173cb000 nid=0x30a4 waiting on condition [0x0000000000000000]
       java.lang.Thread.State: RUNNABLE
    
我们可以看到，上面打印了有关线程的栈轨迹、线程状态。

### jcmd
可以直接用jcmd命令([帮助文档](https://docs.oracle.com/en/java/javase/11/tools/jcmd.html#GUID-59153599-875E-447D-8D98-0078A5778F05)),来
替代前面除了jstat之外的所有命令。

### 小结
本文主要是有关java虚拟机监控的常见命令，以及其左右和怎样使用。一般我们要查看某个进程
的信息(堆栈 线程 GC 性能....)，先要使用jps命令找到对应的pid,然后通过相应的命令以及其子命令去
查看需要对应的信息。

[GUI MAT的使用](http://tryingpfq.top/2019/03/06/matTool/)

参考：[极客时间](https://time.geekbang.org/column/article/40520)
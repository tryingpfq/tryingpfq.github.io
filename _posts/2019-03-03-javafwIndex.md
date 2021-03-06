---
layout:     post
title:      Java架构
subtitle:   Java架构学习内容
date:       2019-03-03
author:     tryingpfq
header-img: img/post-bg-javafw1.jpg
catalog: true
tags:
    - javafw
---

> 本文主要总结java架构师需要学习的内容。是一个总纲目录

### 计算机基础知识

#### 数据结构与算法
   * 1：[剑指Offer](https://www.nowcoder.com/ta/coding-interviews?page=1)
   * 2：[leetcode](https://leetcode.com)

#### 计算机网络
   * 1：书籍《图解http》《图解TCP/IP》《TCP/IP详解卷1》
   * 2：http
   * 3：.........

#### 操作系统
   * 1：进程和线程
   * 2：死锁
   * 3：进程同步 PV 信号量
   * 3：CPU调度算法
   * 4：页面置换算法
   * 5：内存管理方式



### Java

#### 语言基础
   * 1：java的8种基本数据类型，自动装箱和拆箱
   * 2：泛型，以及泛型擦除
   * 3：面向对象特性
   * 4：Stack Quene
   * 5：String StringBuffer StringBuild
   * 6: equils 和 ==
   * 7：关键字（transient intanceof final）
   * 8：try catch finally
   * 9：finalize finalization finally
   * 10：序列化
   * 11：IO
   * 12：静态内部类 和 匿名类
   * 13：发射
   * 14：异常
   * 15：枚举
   * 16：其他常见问题（String.valueOf he Integer.toString 区别）

#### Java -version 新特性
   * 1：Java8 lambda表达式 Stream API 、LomBok包中的注解、Optional
   * 2：Java9 JIgsaw Jshell Reactive Streams
   * 3：Java10 var、 G1的并行Full GC 、TreadLocal握手机制


### 高级篇

#### 锁
   * 1：[volatile](https://blog.csdn.net/tryingpfq/article/details/82142763)
   * 2: [synchronize](https://blog.csdn.net/tryingpfq/article/details/82115612)
   * 3：[CAS](https://blog.csdn.net/tryingpfq/article/details/83177870)
   * 4：ReentrantLock he Lock

#### Java线程
   * 1：创建线程的几种方式，和比较
   * 2：线程状态
   * 3：一般线程和守护线程
   * 4：sleep wait yield notify notifyAll
   * 5：中断线程
   * 6：[死锁](http://tryingpfq.top/2019/03/11/deadlock/)
   * 7：线程如何通信
   * 8：线程池(要做到自己会设计线程池)

#### 网络编程
   * 1：socket
   * 2：protobuf

#### Java并发编程
   * [概述](http://tryingpfq.top/2019/03/05/concurrenSummarize/)
   * [并发编程问题源头](http://tryingpfq.top/2019/03/07/conQuestionSource/)
  
        
#### juc包
jdk中的concurrent包，是java线程底层实现的基础
   * 1：Tools
        * 1：CountDownLatch
        * 2：CyclicBarrier
        * 3：Semaphore
        * 4：Exchanger
   * 2：List Set Map
   * 3：Quene
   * 4：线程池
   * 5：原子变量
        * 1：AtomicInteger and AtomicLong
        * 2：AtomicReference.........


### JVM
   * 1：内存分区
   * 2：JMM内存模型
         * 1：JMM中的happen-before
         * 2：内存模型的底层实现（内存屏障）
         * 3：内存可见性 重排序
   * 3：[垃圾回收](http://tryingpfq.top/2019/03/13/gc/)
        * 1：如何标记垃圾（已经死亡的对象）
        * 2：回收的三种方式（清除，压缩，复制） GC 算法
        * 3：垃圾回收器
        * 4：Minor Gc 和 Full Gc区别
        * 5：回收策略（新生代和老年代）
   * 4：[加载Class类](http://tryingpfq.top/2019/02/15/loadClass/)
   * 5：类加载机制（双亲委派）
   * 6：Java字节码
   * 7：JVM调优
   * 8：JVM参数
   * 9：[JVM性能监控工具与故障处理工具](http://tryingpfq.top/2019/03/06/monitorTools/)


### 设计模式
   * 1：Proxy 代理模式
   * 2：Factory 工厂模式
   * 3：Singleton 单例模式
   * 4：Delegate 委派模式
   * 5：Strategy 策略模式
   * 6：Prototype 原型模式
   * 7：Template 模板模式

### 框架

#### Spring
   概括
   * 1：Application 和 BeanFactory 区别 FacotoryBean 和 BeanFactory
   * 2：Spring Bean 生命周期
   * 3：Spring IOC
   * 4：Spring AOP
   * 5：事物
        * 1：事物实现方式
        * 2：事物传播级别
        * 3：事物的嵌套失效
   * 6：Spring MVC

#### MyBatis
   * 1：关联查询 、嵌套查询
   * 2：缓存
   * 3：Mapper Mybatis的事物
   * 4：分析MyBatis动态代理的真正实现

### 数据库
   * 1：索引 B树和B+树
   * 2：Innodb 和 MyISAM
   * 3：事物隔离级别
   * 4：SQL
   * 5；数据库连接池
   * 6：分库分表
   * 7：数据库锁
   * 8：其他

### Linux


### 分布式
   * 1：分布式架构原理
   * 2：分布式架构策略
        * 1：网络通信原理
        * 2：通信协议中的序列和反序列
        * 3：分析Zookeeper
   * 3：分布式架构中间件
        * 1：RabbitMq消息中间件
        * 2：ActiveMq消息中间件
        * 3：Kafka百万级吞吐
   * 4：缓存和NoSQL
        * 1：Redis高性能缓存数据库
        * 2：Memacached
        * 3：MongoDB
        * 4：Netty
   * 5：实战案例
        * 1：分布式解决方案
        * 2：Session跨越共享 和 分布式单点登录SSO
        * 3：分布式事物解决方案
        * 4：分布式框架下分布式锁的解决方案
        


### 微服务架构
   * 1：SpringBoot
   
   * 2：SpringCloud
   
   * 3：Docker虚拟化技术
   
   * 4：Dubbo

### 高并发
   * 1：分表分库
   * 2：CDN技术
   * 3：消息队列
   * 4：Nginx
   * 5：负载均衡

### 协作开发工具
   * 1：Git
        * 1：git工作原理
        * 2：git常用操作及问题
   * 2：Maven
   
   * 3：Jenkins
   
   * 4:：Sonar

### 其他语言
   * 1：Golang
   * 2：Groovy
   * 3: Python

### 大数据框架
   * 1：spark
   * 2：strom
   * 3：hadoop


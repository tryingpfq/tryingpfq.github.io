---
layout:     post
title:      线程池
subtitle:   不得不懂的线程池
date:       2019-05-15
author:     tryingpfq
header-img: img/post-bg-threadpool.jpg
catalog: true
tags:
    - 并发
---

> 为什么会想到写这篇文章呢，其实网上也挺多的，主要是这次和同学讨论到一个问题，
>就涉及到线程池，而且有些疑问，然我去看一些博客的时候也写的不对，
>最后还是通过看源码解决自己的疑问。首先，这篇博客会分析一些源码，线程池的设计，然后要注意的一些事项。



### 基本概率

#### 线程池是什么

线程池（Thread Pool）是一种基于池化思想管理线程的工具，经常出现在多线程服务器中，如MySQL 数据库连接池。通常，在我们创建线程和销毁线程的时候，都会带来性能的开销。然而线程池维护多个线程，等待监督管理者分配可并发执行的任务。这种做法，一方面避免了处理任务时创建销毁线程开销的代价，另一方面避免了线程数量膨胀导致的过分调度问题，保证了对内核的充分利用。



#### ThreadPoolExecutor

而本文描述线程池是JDK中提供的`ThreadPoolExecutor`类，看一下这个类图关系

![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/post-bg-threadpool1.png?raw=true)

顶层接口`Executor`提供了一种思想：将任务提交和任务执行进行解耦用户无需关注如何创建线程，如何调度线程来执行任务，用户只需提供Runnable对象，将任务的运行逻辑提交到执行器(Executor)中，由`Executor`框架完成线程的调配和任务的执行部分。

`ExecutorService`接口增加了一些能力：

（1）扩充执行任务的能力，补充可以为一个或一批异步任务生成Future的方法。

（2）提供了管控线程池的方法，比如停止线程池的运行。`AbstractExecutorService`则是上层的抽象类，将执		行任务的流程串联了起来，保证下层的实现只需关注一个执行任务的方法即可。

最下层的实现类`ThreadPoolExecutor`实现最复杂的运行部分，`ThreadPoolExecutor`将会一方面维护自身的生命周期，另一方面同时管理线程和任务，使两者良好的结合从而执行并行任务。

`ThreadPoolExecutor`提供了四个构造方法，可以自行进行参数配合，其主要目的的对方便用户根据业务自行控制。



#### 线程池好处

- **降低资源消耗**：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
- **提高响应速度**：任务到达时，无需等待线程创建即可立即执行。
- **提高线程的可管理性**：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- **提供更多更强大的功能**：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池`ScheduledThreadPoolExecutor`，就允许任务延期执行或定期执行。



### 线程池的创建

这里我们还是继续上面那张图吧，对JDK 1.8中`ThreadPoolExecutor`构造方法参数进行分析。

* **corePoolSize**

  核心线程数，默认情况下核心线程会一直存活，即使处于闲置状态也不会受存`keepAliveTime`限制。除非将`allowCoreThreadTimeOut`设置为`true`。(*这里注意下，即使没有任务，核心线程也不会被回收的，这个问题后面再源码分析中会讲到，先注意下*)

* **maximumPoolSize**

  线程池所能容纳的最大线程数。超过这个数的线程将被阻塞。当任务队列为没有设置大小的`LinkedBlockingDeque`时，这个值无效，也就是说如果线程数大于核心了，那么就要放到阻塞队列中去，如果阻塞队列满了的话，才会考虑线程扩容，当超过最大线程数的话，就需要想要的拒绝策略了。(这里也会在源码进行分析，先记住就好

* **keepAliveTime**

  非核心线程的闲置超时时间，超过这个时间就会被回收。

* **unit**

  这个就不多说，就是时间单位。

* **workQueue**

  线程池中的任务队列，常用的有三种队列：`SynchronousQueue`,`LinkedBlockingDeque`,`ArrayBlockingQueue`

  | 队列                    | 描述                                                         |
  | ----------------------- | ------------------------------------------------------------ |
  | `ArrayBlockingQueue`    | 这是一个基于数组阻塞队列，有界的，容量自行传入，按照FIFO的原则 |
  | `LinkedBlockingQueue`   | 基于链表的阻塞队列，按照FIFO原则，有界的，容量可自行传入。注意：这个队列的最大长度是Integer.MAX_VALUE，所以用这个可能有OOM风险。 |
  | `LinkedBlockingDeque`   | 基于链表结构实现的双向阻塞队列                               |
  | `PriorityBlockingQueue` | 支持任务按照优先级排序，默认是自然排序的，当然也可以自定义任务，实现Runnable 和 Comparable接口来自定义排序。 |
  | `DelayQueue`            | 延迟获取队列，在创建元素的时候，可以指定多久才能从队列中获取。 |
  | `SynchronousQueue`      | 一个不存储元素的阻塞队列，每一个put操作必须等待take操作，否则不能添加元素。 |

  

* **threadFactory**

  看名字就很明显，线程工厂，工厂那么肯定是用来创建的了，这里就是用来给这个线程池创建线程的，也就是说，线程池需要并且可以新建线程的时候，就会通过该工厂了进行创建。工厂统一要实现`java.util.concurrent.ThreadFactory`这个接口，可以看下jdk提供的默认线程工厂实现。`io.netty.util.concurrent.DefaultThreadFactory`

  ```java
  //这里要注意的一个东西就是 方法参数的这个r,后面在源码中也会提到
  @Override
  public Thread newThread(Runnable r) {
      	// 这里就是用来创建线程，其实在这个地方，我们可以对线程进行分组命名，方便查看
          Thread t = newThread(new DefaultRunnableDecorator(r), prefix + 								   nextId.incrementAndGet());
          try {
              //是否为守护线程设置
              if (t.isDaemon()) {
                  if (!daemon) {
                      t.setDaemon(false);
                  }
              } else {
                  if (daemon) {
                      t.setDaemon(true);
                  }
              }
              //优先级设置，这个我没去具体分析，猜错应该是线程调度的一个优先级问题
              if (t.getPriority() != priority) {
                  t.setPriority(priority);
              }
          } catch (Exception ignored) {
          }
          return t;
      }
  ```

* **RejectedExecutionHandler**

  拒绝策略，这个是什么时候用到呢，就是线程池任务繁忙，不能再接收任务的时候，就会有相应的拒绝回应。

  如果生产环境下，遇到这种问题，可能就像调整参数，那么该如何去动态修改呢，这是个问题，先留着吧？

  看下`ThreadPoolExecutor`默认提供的拒绝策略。`java.util.concurrent.ThreadPoolExecutor.AbortPolicy`

  ```java
  public static class AbortPolicy implements RejectedExecutionHandler {
        	//这里其实就抛一个异常
          public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
              throw new RejectedExecutionException("Task " + r.toString() +
                                                   " rejected from " +
                                                   e.toString());
          }
      }
  ```

上面是对参数进行解释，ThreadPoolExeutor提供了四个构造方法，所以创建的时候，按照业务实际，去调整相应的参数即可。

这也有一个思考，创建线程池最快的方式是Executors来创建，但为什么不推荐使用呢，而更推荐的方式是`ThreadPoolExecutor`。

> FixedThreadPool和SingleThreadPool:允许请求队列长度为Integer.MAX_VALUE ,这样的话可能导致大量的请求堆积哦，从而导致OOM
>
> CacheThreadPool和ScheduleThreadPool:允许创建线程数量为Integer.MAX_VALUE，可能创建大量线程，从而导致OOM





### 设计与源码剖析

#### 运行机制

上面我们已经提到过，线程池，是以任务(task)的形式进去驱动的，也就是说，是一个任务的接收和处理，既**生产者-消费者** 模型。

说下大概流程吧：后面再做源码分析，只说大致流程，首先有先的任务添加，那么这个线程池就重复addTask操作，先判断当前线程池正在工作的线程数量有没有超过核心线程，如果没有超过的话，就直接创建一个新的线程来处理该任务；如果超过了，就需要考虑任务对是否已经满，如果未满，加入到任务队列中；如果满了，就要看当前工作线程数量和允许的最大线程数，未超过，则创建新的线程来处理该任务，否则执行相应的拒绝策略。

看下流程图：

![](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/post-bg-threadpool2.jpg?raw=true)





以上要感谢 [美团技术博客](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)    [java 多线程](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)
---
layout:     post
title:      线程池
subtitle:   不得不懂的线程池
date:       2020-05-15
author:     tryingpfq
header-img: img/post-bg-threadpool.jpg
catalog: true
tags:
    - 并发
---

> 为什么会想到写这篇文章呢，其实网上也挺多的，主要是这次和同学讨论到一个问题(就是线程工厂创建线程的时候，再次对runnable的run方法捕获异常)，就涉及到线程池，而且有些疑问，然我去看一些博客的时候也写的不对，最后还是通过看源码解决自己的疑问。首先，这篇博客会分析一些源码，线程池的设计，然后要注意的一些事项。如果认真读完，保证有收获的。



### 基本概念

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

  线程池中的任务队列，常用的有三种队列：`SynchronousQueue`,`LinkedBlockingDeque`,`ArrayBlockingQueue`，要注意的是这个是**阻塞**队列，为什么是阻塞呢，其实有用意的，后面再源码中会提到。

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

  看下`ThreadPoolExecutor`默认提供的拒绝策略，`java.util.concurrent.ThreadPoolExecutor.AbortPolicy`

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
  
  线程池还有提供其他三种拒绝策略。代码就不贴了。
  
  1. `AbortPolicy`：直接抛出异常，这是默认策略；
  
  2. `CallerRunsPolicy`：用调用者所在的线程来执行任务；
  
  3. `DiscardOldestPolicy`：丢弃阻塞队列中靠最前的任务，并执行当前任务；
  
  4. `DiscardPolicy`：直接丢弃任务；
  
     

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



要注意的下，线程池中，并没有标记哪些线程为核心线程，只是维护一个核心线程数量，当工作中的线程小于等于这个数量的时候，这些线程都可以称之为核心线程，但超过的时候，就有其他线程，但具体哪些是核心线程，不能明确定义的，这个是个动态的过程，谁都可能成为核心线程状态。



#### 生命周期

线程池，是有生命周期的，但在平时我们用的时候，并没有显示的进行设置，而是由内部来维护的。线程池中存在一些变量俩维护状态，看下几个重要的字段。

```java
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;  // 32-3
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;  //这个就有点大了 可以算一下

    // 五种状态
	//能接受新提交的任务，并且也能处理阻塞队列中的任务；
    private static final int RUNNING    = -1 << COUNT_BITS;

	//关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务。在线程池处于 RUNNING 状态		时，调用 shutdown()方法会使线程池进入到该状态。（finalize() 方法在执行过程中也会调用shutdown()方	法进入该状态）；
    private static final int SHUTDOWN   =  0 << COUNT_BITS;

	//不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。在线程池处于 RUNNING 或 SHUTDOWN 	状态时，调用 shutdownNow() 方法会使线程池进入到该状态；
    private static final int STOP       =  1 << COUNT_BITS;

	//如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 	方法进入TERMINATED 状态。
    private static final int TIDYING    =  2 << COUNT_BITS;

	//在terminated() 方法执行完后进入该状态，默认terminated()方法中什么也没有做。
    private static final int TERMINATED =  3 << COUNT_BITS;

```

* `clt`是对线程池的运行状态和线程池中工作的线程数量进行控制的一个字段。它包含两部分内容：线程池的运行状态(`runState`)和线程池内有效线程的数量 (`workerCount`),这里可以看到，使用了Integer类型来保存，高3位保存`runState`，低29位保存`workerCount`，两个变量之间互不干扰。看下clt相关的方法。

  ```java
  //计算当前运行状态
  private static int runStateOf(int c)     { return c & ~CAPACITY; }
  
  // //计算当前活动线程数量
  private static int workerCountOf(int c)  { return c & CAPACITY; } 
  
   //通过状态和线程数生成ctl
  private static int ctlOf(int rs, int wc) { return rs | wc; }  
  ```



#### 源码

* **Worker**

  这是`ThreadPoolExecutor`的一个内部类，先来简单看一下。`java.util.concurrent.ThreadPoolExecutor.Worker`

  ```java
     private final class Worker extends AbstractQueuedSynchronizer
          implements Runnable
      {
   		// 这个是这个worker持有的真正线程。也是由工厂创建的
         // 要启动这个worker,必须调用这个线程的start方法
          final Thread thread;
        
         //这个是初始创建的时候，赋予这个worker的任务
         //可以为null，如果未null的话，这个worker就会去队列中拿去任务
         //如果不为null的话，这个worker首先会处理这个任务，然后再去阻塞队列中轮询任务
          Runnable firstTask;
        
          Worker(Runnable firstTask) {
              setState(-1); 
              this.firstTask = firstTask;
              //看下工厂方法中，传入的runnable 是这个worker本身自己。
              //所以当这个thread start后，真正执行的是下面这个runWorker(this)方法 这个里面东西后			再重点看吧，毕竟是任务处理的地方
              this.thread = getThreadFactory().newThread(this);
          }
         
          public void run() {
              //这个方法后面分析
              runWorker(this);
          }
      }
  ```

其实上面的贴的代码中的注释，已经说明了，线程池线程创建的一个过程，后面看具体调用地方就好。在创建的时候，先是创建一个Worker,在Worker里面由工厂创建真正的线程thread，要其他这个worker,必须要start 内部创建的thread，当然worker并不止这些，这里只简单说明一下而已。先了解这么多吧，后面具体调用会分析到。

* **execute()方法**

  `java.util.concurrent.ThreadPoolExecutor#execute`，这个是任务提交的基础类，这个是没有执行结果返回的，这里就只分析这个吧，submit()方法最终也会调用这个。

  ```java
  public void execute(Runnable command) {
          if (command == null)  // 任务不允许为空
              throw new NullPointerException();
      
      	// 获取当前ctl的值，这个值维护这状态和当前工作线程数量
          int c = ctl.get();
      
          /*
           * workerCountOf方法取出低29位的值，表示当前活动的线程数；
           * 如果当前活动线程数小于核心线程corePoolSize，则新建一个线程放入线程池中
           * 并把任务添加到该线程中。(注意，这里再创建核心线程的时候，worker中的runnable是不为null			 * 的)
           */
          if (workerCountOf(c) < corePoolSize) {
           /*
           * 看下addWorke方法的第二个参数，true false 标示是否为核心线程创建
           */
              if (addWorker(command, true))
                  return;		//成功直接放回
              // 失败的话 要重新获取这个状态值
              c = ctl.get();
          }
      	 /*
           * 如果运行到这里，首先说明一个问题，就是此时工作中的线程，已经超过核心线程了。
           * 所以要先判断是否为运行状态，并且阻塞队列未满。就会进入下面这个if
           */
          if (isRunning(c) && workQueue.offer(command)) {	
              //再次重新获取ctl状态为值
              int recheck = ctl.get();
               /*
               * 再次判断线程池的运行状态，如果不是运行状态，
               * 由于之前已经把command添加到workQueue中了，这时需要移除该command
               
               * 这时候就是添加任务失败，
               * 就需要执行拒绝策略对该任务进行处理，整个方法返回
               * 这里为什么会出现这个问题，可能是担心其他线程对这个线程池状态有修改。
               */
              if (! isRunning(recheck) && remove(command))	//这里面其实是双重检查
                  reject(command);
           
           /* 执行到这里，说明任务已经条件到阻塞队列中去了，上面移除失败了
           * 获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
           * 要注意下这里方法传入的参数
           * 1. 第一个参数为null，表示在线程池中创建一个线程，不设置当前任务。
           * 2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根		  *	据maximumPoolSize来判断；
           * 如果判断workerCount大于0，则直接返回，说明有工作线程
           * 在workQueue中新增的command会在将来的某个时刻被执行。
           */
              else if (workerCountOf(recheck) == 0)//这里面其实是双重检查
                  addWorker(null, false);
          }
          /*
           * 如果执行到这里，存在下面两种情况
           * 1. 线程池已经不是RUNNING状态
           * 2. 线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
           * 这时，再次调用addWorker方法，第一个参数是command,如果创建成功，则会在创建的线程直接执		   * 行，但第二个参数传入为false。
           * 创建失败的话，就进入拒绝策略。（比如，队列已满，且大于最大线程了，这个就会失败的）
           */
          else if (!addWorker(command, false))
              reject(command);
      }
  
  ```

  上面已经分析到，对于新的任务申请，判断一个创建的流程，和之前的流程图是一样的，可以回到上面去看一下，其实就是通过判断线程池状态和工作中的线程数量，还有就是判断核心和最大线程数，已经阻塞队列是否已经满，但要注意的是上面的`addWoker(Runnable,boolen)`，这个方法里面的两个参数，第一个什么时候为`null`，第二个什么时候是`true`和`false`。很明显，接下来就是`addWorker()`这个方法了。

  

* **addWorker**

  `java.util.concurrent.ThreadPoolExecutor#addWorker`

  ```java
  private boolean addWorker(Runnable firstTask, boolean core) {
          retry:
          for (;;) {
              int c = ctl.get();
              int rs = runStateOf(c);
  
               /*
               * 首先是判断 rs >= SHUTDOWN，则表示此时不再接收新任务；
               * 接着判断以下3个条件，只要有1个不满足，则返回false：
               * 1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞				 * 队列中已保存的任务，这个在上面线程状态中有说明过。
               * 2. firsTask为空
               * 3. 阻塞队列不为空
               * 
             	 * 这里的判断要综合execute方法一起看。
             	 * 主要分析这部分
             	 * @code{
             	 * ! (rs == SHUTDOWN &&
               *      firstTask == null &&
               *     ! workQueue.isEmpty())
             	 * }
             	 * 如果第一个条件为真，这个时候本身就不能接受条件的， 这个时候firstTask != null肯定不			  * 行的
             	 
             	 * 如果firstTask为空，并且workQueue也为空，则返回false，
           	 * 因为队列中已经没有任务了，不需要再添加线程了
          	 */
              if (rs >= SHUTDOWN &&
                  ! (rs == SHUTDOWN &&
                     firstTask == null &&
                     ! workQueue.isEmpty()))
                  return false;
  
              for (;;) {
                  // 获取当前线程数
                  int wc = workerCountOf(c);
        
                  /*
                   * 如果超过最大值，这个就是ctl的低29位，很大的。则直接返回
                   * 第二个判断就是前面是怎样来创建worker的，并和相应的线程数量比较
                   * 也就是核心和最大，不符合就返回false
          		 */
                  if (wc >= CAPACITY ||
                      wc >= (core ? corePoolSize : maximumPoolSize))
                      return false;
                  // 尝试增加workerCount，如果成功，则跳出第一个for循环
                  if (compareAndIncrementWorkerCount(c))
                      break retry;
                  // 如果增加workerCount失败，则重新获取ctl的值
                  c = ctl.get();
                  // 再次判断状态是否有改变，有修改的话，返回第一个for循环继续执行
                  // 这里个前一个判断 我觉得应该是以防这时候其他线程，有对线程池状态修改
                  if (runStateOf(c) != rs)
                      continue retry; 
              }
          }
          boolean workerStarted = false; boolean workerAdded = false;
          Worker w = null;
          try {
              //这里再分析worker的时候有分析，可以看上面
              w = new Worker(firstTask);
              // 此时的这个t 就是线程工厂创建的，不理解就看上面Worker
              final Thread t = w.thread;
              if (t != null) {
                  final ReentrantLock mainLock = this.mainLock;
                  mainLock.lock();
                  try {
             			//获取状态
                      int rs = runStateOf(ctl.get());
  					//只有在状态符合的情况下 才会把这个worker添加到workers中。
                      if (rs < SHUTDOWN ||
                          (rs == SHUTDOWN && firstTask == null)) {
                          if (t.isAlive()) // precheck that t is startable
                              throw new IllegalThreadStateException();
                          workers.add(w);
                          int s = workers.size();
                          if (s > largestPoolSize)
                              largestPoolSize = s;
                          workerAdded = true;
                      }
                  } finally {
                      mainLock.unlock();
                  }
                  if (workerAdded) {
                      // 添加成功，启动这个worker,其实是启动worker中创建的线程
                      // 然后执行里面的run方法。
                      t.start();
                      workerStarted = true;
                  }
              }
          } finally {
              if (! workerStarted)
                  //如果失败 要尝试移除
                  addWorkerFailed(w); 
          }
          return workerStarted;
      }
  
  ```

  上面把addWorker方法分析了一下，这里的判断要和前面execute综合来看。其实这里总结来说，就是判断能不能创建worker,可以的话就创建，并放到workers中去和启动线程。接下来就是要看。Worker中的run中的runWorker方法了

* **runWorker**

  `java.util.concurrent.ThreadPoolExecutor#runWorker`,这里面是真正消耗任务的地方。

  ```java
  final void runWorker(Worker w) {
          Thread wt = Thread.currentThread();
      	// 初始执行，获取初始化的任务，上面说过，这个任务可能为null
          Runnable task = w.firstTask;
          w.firstTask = null;
          w.unlock();
      	//这个标记 是用来标记是否异常而退出循环
      	// 如果task.run()中有异常没有捕获
          boolean completedAbruptly = true;
          try {
              //这是个while循环，来轮询阻塞队列中的任务
              //getTask()方法后面分析
              while (task != null || (task = getTask()) != null) {
                  w.lock();
                  if ((runStateAtLeast(ctl.get(), STOP) ||
                       (Thread.interrupted() &&
                        runStateAtLeast(ctl.get(), STOP))) &&
                      !wt.isInterrupted())
                      wt.interrupt();
                  try {
                      // 执行前置处理
                      // ThreadPoolExetutor是没有实现的，其实就是一个钩子
                      beforeExecute(wt, task);
                      Throwable thrown = null;
                      try {
                          task.run();
                      } catch (RuntimeException x) {
                          // 捕获后重新抛出 会在最外层finnally执行
                          thrown = x; throw x;
                      } catch (Error x) {
                          thrown = x; throw x;
                      } catch (Throwable x) {
                          thrown = x; throw new Error(x);
                      } finally {
                          //执行后置处理
                          //ThreadPoolExetutor是没有实现的，其实就是一个钩子
                          afterExecute(task, thrown);
                      }
                  } finally {
                      task = null;
                      w.completedTasks++;
                      w.unlock();
                  }
              }
              completedAbruptly = false;
          } finally {
              //线程退出，这里可能是正常或者异常退出
              processWorkerExit(w, completedAbruptly);
          }
      }
  
  ```

  上面就对runWorker方法分析了一下，总结一下就是：

  >1. while循环不断地通过getTask()方法获取任务；
  >2. getTask()方法从阻塞队列中取任务；
  >3. 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态；
  >4. 调用`task.run()`执行任务；
  >5. 如果task为null则跳出循环，执行processWorkerExit()方法；
  >6. runWorker方法执行完毕，也就是退出while循环，那么说明阻塞队列中不再有任务，线程退出，销毁。

  接下来看getTask()这个方法。

* **getTask**

  `java.util.concurrent.ThreadPoolExecutor#getTask`,这个方法顾名思义就是到阻塞队列中去获取任务。

  ```java
  private Runnable getTask() {
      	// 表示去任务是否超时
          boolean timedOut = false;
  
          for (;;) {
              int c = ctl.get();
              int rs = runStateOf(c);
  
               /*
               * 如果线程池状态rs >= SHUTDOWN，也就是非RUNNING状态，再进行以下判断这个时候，队列中			   * 是不允许添加任务的
               * 1. rs >= STOP:线程池是否正在stop；
               * 2. 阻塞队列是否为空。
               * 如果以上条件满足，则将workerCount减1并返回null,也就是运行的线程数要减少，这个线程			 * 会退出的
          	 */
              if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                  decrementWorkerCount();
                  return null;
              }
  			
              // 获取线程数
              int wc = workerCountOf(c);
  
              /*
               * 首先timed是用来表示去队列中获取任务是否阻塞，而不是超时阻塞
           	 * allowCoreThreadTimeOut默认是false，也就是核心线程不允许进行超时；
          	 *	wc > corePoolSize，表示当前线程池中的线程数量大于核心线程数量；
         		 *  对于超过核心线程数量的这些线程，需要进行超时控制
               */
              boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
  
               /*
           	 * wc > maximumPoolSize的情况是因为可能在此方法执行阶段有线程修改最大线程数     
               * timed && timedOut 如果为true，表示当前操作需要进行超时控制，
               * 并且上次从阻塞队列中获取任务发生了超时
               * 接下来判断，如果有效线程数量大于1，或者阻塞队列是空的，那么尝试将workerCount减1；
               * 如果减1失败，则返回重试,也就是这里要。
               * 如果wc == 1时，也就说明当前线程是线程池中唯一的一个线程了。
               * 如果返回为null，这线程就会退出。
               * 
               * 这个地方注意下，下面专门俩解释一下(1)
               */
              if ((wc > maximumPoolSize || (timed && timedOut))
                  && (wc > 1 || workQueue.isEmpty())) {
                  if (compareAndDecrementWorkerCount(c))
                      return null;
                  continue;
              }
  
              try {
                  Runnable r = timed ?
                      workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                      workQueue.take();
                  if (r != null)
                      return r;
                  timedOut = true;
              } catch (InterruptedException retry) {
                  timedOut = false;
              }
          }
      }
  ```

  上面就是从队列中获取任务的一个过程，这里解释下上面(1)那个地方：

  >这里如果timed = false,也就是线程小于等于核心，那么就不会进入下面这if判断，因为核心
  >
  >是不会大于最大的。然后就要进入下面的try中，此时timed=false,那么`workQueue.take()`这个方法就会阻塞，直到有新的任务生产。这里也就解释了为什么队列要是阻塞队列，以及线程池如如何保证核心线
  >
  >不会被回收，网上有些博客说会回收核心线程，还有的说不会是因为会在这里无限循环，一定要自己看源码。*当然我认为，核心线程还是可能会被回收的，不过是因为异常而回收，加入这个时候都阻塞在这里，来了一个任务，那么就会唤醒(这里有个疑问，是唤醒所有还是单个呢，觉得应该是单个)，加入这个任务里面抛异常了没处理，就会调用上面说的`processWorkerExit`这个方法，异常退出*（这里理解错误了，后面会解释）。allowCoreThreadTimeOut默认是false，但如果设置为true的话，timed就会为true，核心线程就会被回收。

  上面也对线程回收做了一个解释。

  

  #### worker的回收

  什么时候会销毁？当然是`runWorker`方法执行完之后，也就是Worker中的run方法执行完，退出while循环，然后就会从workers中移除，也就不再引用，由JVM自动回收。

  `getTask`方法返回null时，在`runWorker`方法中会跳出while循环，或者task抛异常，然后会执行`processWorkerExit`方法。接下来肯定是`processWorkerExit`这个方法了。

  

* **processWorkerExit**

  `java.util.concurrent.ThreadPoolExecutor#processWorkerExit`,线程退出。

  ```java
  private void processWorkerExit(Worker w, boolean completedAbruptly) {
          // 如果completedAbruptly值为true，表示是task中抛异常了，没有捕获，那么需要将
          //workerCount减1
      
          // 如果线程执行时没有出现异常，既m没有任务了，getTask()方法中已经已经对workerCount进行了		//减1操作，		
      	//这里就不必再减了。  
          if (completedAbruptly)
              decrementWorkerCount();
  
          final ReentrantLock mainLock = this.mainLock;
          mainLock.lock();
          try {
              //统计完成的任务数，然后从workers中移除这个worker
              completedTaskCount += w.completedTasks;
              workers.remove(w);
          } finally {
              mainLock.unlock();
          }
  
      	//根据线程池状态进行判断是否结束线程池
          tryTerminate();
  
          int c = ctl.get();
           /*
           * 当线程池是RUNNING或SHUTDOWN状态时，如果worker是异常结束，那么会直接addWorker
           * 注意这里，上面我说到一个地方有误，就是我说的核心线程会退出的情况，其实是会退出，但会重新
           * 构建一个线程。
           
           * 如果allowCoreThreadTimeOut=true，并且等待队列有任务，至少保留一个worker；
           * 如果allowCoreThreadTimeOut=false，workerCount不少于corePoolSize。
           * 
           * 也就是说 如果线程状态是RUNNING 或者SHUTDOWN 就会进入下面方法
           */
          if (runStateLessThan(c, STOP)) {
              if (!completedAbruptly) {
                  // 若允许核心线程超时回收，则最低线程数量为0，否则为核心线程数
                  int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                  if (min == 0 && ! workQueue.isEmpty())
                      min = 1;
                  
                  if (workerCountOf(c) >= min)
                      return; // replacement not needed
              }
              // 这里说明线程是异常引起，就会重构线程放入线程池，
              //或者上面当前线程小于最低线程数，也会添加
              addWorker(null, false);
          }
      }
  
  ```

  以上差不多就是线程的这个生命周期，其他方法目前就不再做分析了，比如`tryTerminate`、`shutdown`，`shutdownNow`

* **Question**

  线程池中，在空闲状态的情况下，最少会保留多少个核心线程？

  *这个看完上面应该就知道，如果`allowCoreThreadTimeOut`=true的话，就会对核心线程进行回收，然后`processWorkerExit`方法会进行判断，至少要保留一个核心线程。*理解错误，即使*这个看完上面应该就知道，如果`allowCoreThreadTimeOut`=true，但没有任务了的话，核心线程应该都会被回收的。



### TODO

线程池监控

动态修改线程池线程数量



### 结尾

相信如果能看到这里，至少对线程池会有个基本了解和任务，以后使用的时候也可以避免一些错误，毕竟线程池的使用，在日常开发是不可避免的，而且这不仅仅是一种技术，我觉得更是解决问题的一种思想。

以上要部分内容来自 [美团技术博客](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)    [深入理解java线程池](https://juejin.im/entry/58fada5d570c350058d3aaad)  必须感谢。
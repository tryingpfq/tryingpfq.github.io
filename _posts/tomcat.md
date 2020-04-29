



### 源码构建

[参考](https://my.oschina.net/u/1018146/blog/1524309)

启动参数

-Dcatalina.home=C:\D \sources\tomcat
-Dcatalina.base=C:\D\sources\tomcat
-Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
-Djava.util.logging.config.file=C:\D\sources\tomcat\conf\logging.properties





### 服务端启动流程

 //TODO 



### 客户端请求过程

从本质上讲，客户端连接到服务端监听的Socket端口，然后通过处理，把返回数据协议Socket输出流。

看了下大概流程，然后顺便记录一下笔记吧：

* `org.apache.tomcat.util.net.NioEndpoint`

  这个怎么理解呢，其实就是用来处理客户端的请求的，在tomcat启动的时候，会对http端口进行监听，具体看bind()方法就好。端口绑定后，会启动poller线程和Acceptor线程。

  ```java
  @Override
      public void startInternal() throws Exception {
  
          if (!running) {
           	//上面的就全部注释掉了  直接看这里
              // Start poller thread
              poller = new Poller();
              /**
               * 以守护线程的形式启动这个poll线程
               */
              Thread pollerThread = new Thread(poller, getName() + "-Poller");
              pollerThread.setPriority(threadPriority);
              pollerThread.setDaemon(true);
              pollerThread.start();
  
              //启动acceptor线程
              startAcceptorThread();
          }
      }
  
  // org.apache.tomcat.util.net.Acceptor#run  只看核心代码
     @Override
      public void run() {
          while (endpoint.isRunning()) {
            
              state = AcceptorState.RUNNING;
              try {
                  try {
                      /**
                       * 这里是阻塞的，只有等有连接来了后，才会被唤醒 {@link ServerSocketChannelImpl#accept0(java.io.FileDescriptor, java.io.FileDescriptor, java.net.InetSocketAddress[])}
                       */
                      socket = endpoint.serverSocketAccept();
                  } catch (Exception ioe) {
                     
          state = AcceptorState.ENDED;
      }
                  
   //org.apache.tomcat.util.net.NioEndpoint.Poller#run 
     @Override
          public void run() {
              while (true) {
                  boolean hasEvents = false;
                  // 这里是对客户端请求的监听，服务器启动后，
                  //这里会一直循环的轮行，判断是否有新的事件
                  try {
                      if (!close) {
                          hasEvents = events();
                          if (wakeupCounter.getAndSet(-1) > 0) {
                              keyCount = selector.selectNow();
                          } else {
                              /**这个地方是个阻塞过程，但有个超时机制，这里默认是1秒
                               * 这里其实可以了解下这selector,这个首先类是SelectorImpl
                               * windows来说，真正的实现{@link WindowsSelectorImpl#doSelect(long)}
                               * 底层是poll模型，调用的是本地方法，应该就是调用操作系统底层API{@link WindowsSelectorImpl.SubSelector#poll0(long, int, int[], int[], int[], long)}
                               * 如果会有新的事件，那么keyCount就会大于0
                               */
                              keyCount = selector.select(selectorTimeout);
                          }
                          wakeupCounter.set(0);
                      }
                  } catch (Throwable x) {
                  }
                  if (keyCount == 0) {
                      hasEvents = (hasEvents | events());
                  }
  
                  Iterator<SelectionKey> iterator =
                      keyCount > 0 ? selector.selectedKeys().iterator() : null;
                  while (iterator != null && iterator.hasNext()) {
                      SelectionKey sk = iterator.next();
                      NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                      if (socketWrapper == null) {
                          iterator.remove();
                      } else {
                          iterator.remove();
                          /**
                           * 处理事件 (1)
                           */
                          processKey(sk, socketWrapper);
                      }
                  }
                  timeout(keyCount,hasEvents);
              }
              getStopLatch().countDown();
          }
       
  ```

  

* `org.apache.tomcat.util.net.AbstractEndpoint#processSocket` 接着看这个方法

  ```java
  public boolean processSocket(SocketWrapperBase<S> socketWrapper,
              SocketEvent event, boolean dispatch) {
          try {
              if (socketWrapper == null) {
                  return false;
              }
              SocketProcessorBase<S> sc = null;
              if (processorCache != null) {
                  sc = processorCache.pop();
              }
              if (sc == null) {
                  /**
                   * 为客户端创建创建Socket处理器
                   * 问题来了，这个是否可以被循环利用呢 上面这个缓存是用力啊干嘛的呢 TODO
                   * 其实这里就是创建一个任务线程
                   */
                  sc = createSocketProcessor(socketWrapper, event);
              } else {
                  sc.reset(socketWrapper, event);
              }
              Executor executor = getExecutor();
              if (dispatch && executor != null) {
                  /**
                   * 把任务提交给线程池，但这里并没有保证线程安全吧，但好像业务处理都会在上面
                   * 创建的sc中一次性处理
                   */
                  executor.execute(sc);
              } else {
                  sc.run();
              }
          } catch (RejectedExecutionException ree) {
              return false;
          } catch (Throwable t) {
       
              return false;
          }
          return true;
      }
  
  ```

* `org.apache.tomcat.util.net.NioEndpoint.SocketProcessor#doRun` 接着看任务线程run方法

  ```java
  @Override
          protected void doRun() {
              /**
               * 获取客户端连接的socket
               */
              NioChannel socket = socketWrapper.getSocket();
              Poller poller = NioEndpoint.this.poller;
              if (poller == null) {
                  socketWrapper.close();
                  return;
              }
          } else {
              /**
               * 处理事件 process 这是核心方法
               */
              state = getHandler().process(socketWrapper, event);
          }
                      
      }
  ```



* 这个调用链太深了，具体断点看吧，但后面会进入到`StandardWrapper`中

  首先看load 方法，这里面做的是什么呢，就是tomcat启动的时候，每个wrapper都会厨师会对应的servlert,

  当然，也可能调用setServlet()方法。比如DispatcherServlet

  ```java
    /**
       * 对于web来说，tomcat会统一 把请求派发给 {@link DispatcherServlet}
       * 然后上层交给 dispatcherServlet 进行任务分发
       * 这里dispatcherServelt会注册到tomcat中对应的wrapper中
       */
      @Override
      public void setServlet(Servlet servlet) {
          instance = servlet;
      }
  ```

开始会进入`org.apache.catalina.core.StandardWrapperValve#invoke`这个方法

```java
//主要看里面三个地方吧
	/**
        * 获取servlet 
       */
     servlet = wrapper.allocate();

  	/**
      * 构建Filter链 并且会把servlet注册到Chain中
     */
     ApplicationFilterChain filterChain =
                ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

	/**
      * 执行Filter 看到这个责任链应该是熟悉了，肯定是过滤完后，会调用servelt的service()方法
      */
	filterChain.doFilter(request.getRequest(),
                                    response.getResponse());
```



最后就会交给上层Servlet中，当然，对于web来说，tomcat是先派发给DispatcherServlet,然后由在分发。

请求流程就先看这么多了，里面东西太多了，有没有发现，其实用Netty是完全可以做的，而且对于线程处理，应该会更优一些。



### catalina


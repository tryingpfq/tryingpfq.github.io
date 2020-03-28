---
layout:     post
title:      Zookeeper
subtitle:   Zookeeper
date:       2019-07-14
author:     tryingpfq
header-img: img/post-bg-zookeeper.jpg
catalog: true
tags:
    - Zookeeper
---

### 初识Zookeeper

### 前言
>现在在分布式当中，基本离不开zookeeper的使用。分布式环境下，主要的特点的分布性、并发性、无序性，自然也会面临一些问题，比如（网络通信：网络本身的不可靠性，因此会涉及到一些网络通信的问题；网络分区：也就是我们通常说的脑裂，当网络发生异常导致分布式系统中部分节点之间的网络延时不断增大，最终导致组成分布式架构的所有节点，只有部分节点能够正常通信；三态：在分布式架构中，就三种状态，成功、失败和超时；分布式事务：ACID，原子性、一致性、隔离性、持久性）。

**看完这篇文章，可以对zk有些感性的认识，比如单机、集群如何安装和搭建，一些基本概念，操作命令，客户端连接，leader选举过程，已近一些应用场景和实现过程，ZAB协议。**



### 中心化和去中心化

假如在分布式系统中，所有节点都围绕一个中心节点去通信或者数据同步，那么假如当这个节点挂掉的时候，这个分布式系统就蹦了。所以在分布式中，每一个节点都是高度自治的，节点之间可以自由连接，任何一个节点都可能成为阶段性的中心，但不具备强制性的中性控制能力。分布式架构里面，很多的架构思想采用的是：当集群发生故障的时候，集群中的人群会自动“选举”出一个新的领导。
最典型的是： zookeeper / etcd


### CAP/BASE理论
**CAP：在分布式中，不管如何，同时只能满足两种条件。CP or AP** 

 * C（一致性 Consistency）: 所有节点上的数据，时刻保持一致
 * A可用性（Availability）：每个请求都能够收到一个响应，无论响应成功或者失败
 * 分区容错 （Partition-tolerance）：表示系统出现脑裂以后，可能导致某些server与集群中的其他机器失去联系

CAP理论仅适用于原子读写的Nosql场景，不适用于数据库系统

**BASE**

基于CAP理论，CAP理论并不适用于数据库事务（因为更新一些错误的数据而导致数据出现紊乱，无论什么样的数据库高可用方案都是徒劳），虽然XA事务可以保证数据库在分布式系统下的ACID特性，但是会带来性能方面的影响；eBay尝试了一种完全不同的套路，放宽了对事务ACID的要求。提出了BASE理论Basically available ： 数据库采用分片模式， 把100W的用户数据分布在5个实例上。如果破坏了其中一个实例，仍然可以保证80%的用户可用

soft-state：在基于client-server模式的系统中，server端是否有状态，决定了系统是否具备良好的水平扩展、负载均衡、故障恢复等特性。
Server端承诺会维护client端状态数据，这个状态仅仅维持一小段时间, 这段时间以后，server端就会丢弃这个状态，恢复正常状态

Eventually consistent：数据的最终一致性



### 基本概念

zookeeper并不是用来存储数据的，通过监控数据状态的变化，达到基于数据的集群管理。

zk是一个典型的分布式协调框架，具有分布式数据一致性的解决方案。它主要用在数据的发布/订阅（配置中心:disconf）、 负载均衡（dubbo利用了zookeeper机制实现负载均衡） 、命名服务、master选举(kafka、hadoop、hbase)、分布式队列、分布式锁。zookeeper的特性，数据一致性：从同一个客户端发起的事物请求，最终会严格按照顺序应用到zookeeper中，原子性：所有的事务请求的处理结果在整个集群中的所有机器上的应用情况是一致的，也就是说，要么整个集群中的所有机器都成功应用了某一事务、要么全都不应用，可靠性：一旦服务器成功应用了某一个事务数据，并且对客户端做了响应，那么这个数据在整个集群中一定是同步并且保留下来的，实时性：一旦一个事务被成功应用，客户端就能够立即从服务器端读取到事务变更后的最新数据状态；（zookeeper仅仅保证在一定时间内，近实时）。


### zookeeper的安装
[下载zookeeper](http://apache.fayea.com/zookeeper/zookeeper-3.5.5/) ,[下载](http://mirrors.hust.edu.cn/apache/zookeeper/stable/)  目前版本是3.5.5,下面是基于Linux的安装，可以使用虚拟机，用Xsheel连接，也方便进行集群搭建。

#### 单机搭建

* 1.	解压zookeeper tar -zxvf zookeeper-3.4.10.tar.gz
* 2.	cd到 ZK_HOME/conf  , copy一份zoo.cfg
* 3.    cp  zoo_sample.cfg  zoo.cfg
* 4. sh zkServer.sh xxx 查看命令 {start|start-foreground|stop|restart|status|upgrade|print-cmd}

     启动命令 sh zkServer.sh start
* 5.	sh zkCli.sh -server  ip:port

#### 集群搭建

1：修改zoo.cfg
	server.id=ip:port:port
	server.1=192.168..129:2888:3181   

​	2888表示follower节点与leader节点交换信息的端口号 3181  如果leader节点挂掉了, 需要一个端口来重新选举。
​	server.2=192.168.146.128:2888:3181   
​	server.3=192.168.146.129:2888:3181

2：zoo.cfg中有一个dataDir = /tmp/zookeeper
$dataDir/myid 添加一个myid文件。这个myid是用来表示自己服务器在集群中是唯一的。范围是1-255



3：启动服务	
	如果需要增加observer节点
	zoo.cfg中 增加 ;peerType=observer
	server.1=192.168.146.128:2888:3181  
	server.2=192.168.146.129:2888:3181   
	server.3=192.168.146.130:2888:3181:observer

zookeeper集群, 包含三种角色: leader / follower /observer

**observer**

observer 是一种特殊的zookeeper节点。可以帮助解决zookeeper的扩展性（如果大量客户端访问我们zookeeper集群，需要增加zookeeper集群机器数量。从而增加zookeeper集群的性能。 导致zookeeper写性能下降， zookeeper的数据变更需要半数以上服务器投票通过。造成网络消耗增加投票成本）

* 1.	observer不参与投票。 只接收投票结果。
* 2.	不属于zookeeper的关键部位。



### zoo.cfg

* tickTime=2000 zookeeper中最小的时间单位长度 （ms）
* initLimit=10follower节点启动后与leader节点完成数据同步的时间
* syncLimit=5leader节点和follower节点进行心跳检测的最大延时时间
* dataDir=/tmp/zookeeper表示zookeeper服务器存储快照文件的目录
* dataLogDir表示配置 zookeeper事务日志的存储路径，默认指定在dataDir目录下
* clientPort表示客户端和服务端建立连接的端口号： 2181



### 基本模型

zookeeper的数据模型和文件系统类似，每一个节点称为：znode. 是zookeeper中的最小数据单元。每一个znode上都可以保存数据和挂载子节点。 从而构成一个层次化的属性结构

**节点特性**

一般在分布式语境下的节点是指组成集群的每一台服务器，在 zookeeper 中还有另外一层意思，称之为数据节点（ZNode）。 zookeeper 的整个名字空间的结构是层次化的，和 Linux 文件系统结构相似，是一颗树。名字空间的层次由斜杠（/）来进行分割，在名称空间里面的每一个节点的名字空间由这个结点的路径来确定。 每个 ZNode 上都会保存自己的数据内容，同时还会保存一系列属性信息。

`持久化节点` ： 节点创建后会一直存在zookeeper服务器上，直到主动删除

`持久化有序节点` ：每个节点都会为它的一级子节点维护一个顺序

``临时节点`` ： 临时节点的生命周期和客户端的**会话**保持一致。当客户端会话失效，该节点自动清理

```临时有序节点``` ： 在临时节点上多勒一个顺序性特性

**Watcher**

**zookeeper**提供了分布式数据发布/订阅,zookeeper允许客户端向服务器注册一个watcher监听。当服务器端的节点触发指定事件的时候会触发watcher。服务端会向客户端发送一个事件通知watcher的通知是一次性，一旦触发一次通知后，该watcher就失效。

**ACL**

zookeeper提供控制节点访问权限的功能，用于有效的保证zookeeper中数据的安全性。避免误操作而导致系统出现重大事故。CREATE /READ/WRITE/DELETE/ADMIN

**版本**

对于每个 ZNode ，zookeeper 都会为它维护一个叫 Stat 的数据结构，Stat 中记录了 ZNode 的三个数据版本，version（当前ZNode数据内容的版本号）、cversion（当前ZNode子节点的版本号）、aversion（当前ZNode的ACL变更版本号）

**会话**

zookeeper 中客户端启动时会与服务器建立一个 TCP 连接，从第一次连接建立开始，客户端会话的生命周期就开始了，通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，也能够向服务器发送请求并接受响应，还能够接收来自服务器的 watch 事件通知。

### 命令操作

* create [-s] [-e] path data acl

  -s表示节点是否有序

  -e表示是否为临时节点

  默认情况下，是持久化节点

*  get path [watch]

  获得指定 path的信息

* set path data [version]

  修改节点 path对应的data

  乐观锁的概念

  数据库里面有一个 version字段去控制数据行的版本号

* delete path [version]

  删除节点

* stat信息

  cversion = 0子节点的版本号

  aclVersion = 0表示acl的版本号，修改节点权限

  dataVersion = 1表示的是当前节点数据的版本号 

  czxid节点被创建时的事务ID

  mzxid节点最后一次被更新的事务ID

  pzxid当前节点下的子节点最后一次被修改时的事务ID
  
  

### Java连接Zookeeper ###

java连接zookeeper主要有三种方式，`zookeeper原始API`，`ZkClien连接`，`Curator`

zookeeper原生API连接：com.tryingpfq.fenbushi.zk.demo.ClienDemo

~~~java
public class ClienDemo implements Watcher {
    private static ZooKeeper zookeeper;

    private static final String Addr = "192.168.48.128:2181,192.168.48.129:2181,192.168.48.130:2181";

    private CountDownLatch countDownLatch = new CountDownLatch(1);

    @Override
    public void process(WatchedEvent watchedEvent) {
        if (watchedEvent.getState() == SyncConnected) {
            System.err.println("watch received event");
            countDownLatch.countDown();
        }
    }

    /**
     * 连接zookeeper
     */
    public  void connectZookeeper()  {
        try {
            zookeeper = new ZooKeeper(Addr, 2000, this);
        } catch (IOException e) {
            e.printStackTrace();
        }
        countDownLatch.countDown();
        System.err.println("zk connection success");
    }

    /**
     * 创建节点
     */
    public String createNode(String path, String data) {
        try {
            return zookeeper.create(path, data.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "";
    }

    public List<String> getChildren(String path) {
        List<String> children = new ArrayList<>();
        try {
             children = zookeeper.getChildren(path, false);
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return children;
    }

    /**
     * 获取节点数据
     */
    public String getNodeData(String path) {
        byte[] data = new byte[0];
        try {
            data = zookeeper.getData(path, false, null);
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        if (data == null) {
            return "";
        }
        return new String(data);
    }

    /**
     * 设置节点信息
     */
    public Stat setData(String path, String data) {
        Stat stat = null;
        try {
            stat = zookeeper.setData(path, data.getBytes(), -1);
        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return stat;
    }

    public void deleteNode(String path) {
        try {
            zookeeper.delete(path, -1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }



    public static void main(String[] args) {
        ClienDemo watcher = new ClienDemo();
        watcher.connectZookeeper();
        String aTry = watcher.createNode("/fuyou","321");
        System.err.println("atry " + aTry);
//        List<String> children = watcher.getChildren("/");
//        System.err.println(children);
    }
}

~~~



### 应用场景

* 统一命名服务

* 配置管理

  实现配置中心有两种模式：push、pull。

  长轮训：zookeeper采用的是推拉相结合的方式。 客户端向服务器端注册自己需要关注的节点。一旦节点数据发生变化，那么服务器端就会向客户端发送watcher事件通知。客户端收到通知后，主动到服务器端获取更新后的数据

  

* 分布式锁

  分布式锁，通常的一些实现方式,`redis`,`数据库`(创建一个表，通过唯一索引方式)

  zookeeper实现

  * **排他锁**

    **实现原理：**

    利用Zookeeper不能重复创建一个节点的特性。可以在zookeeper服务端创建`/Lock`节点，然后所有客户端需要获取锁的时候在改节点下创建lock节点，如果创建成功，那么说明获取锁成功，释放的时候，把这个节点删除，其他客户端就可以再次竞争该锁。

    ![排它锁](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-zk1.jpg?raw=true)

    **代码实现：**基于javaAPI简单 实现

    ```java
    public class LockOne {
    
        private static final String LOCK_PATH = "/Locks/LockOne";
    
        private static ZkCliDemo zkCli;
    
        private static ZooKeeper zk;
    
        public LockOne(){
            zkCli = new ZkCliDemo();
        }
    
    
        public void connect(){
            try {
                zk = zkCli.connectZookeeper();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        /**
         * 获取锁
         * @return
         */
        public boolean lock(){
            if (zk == null) {
                return false;
            }
            CountDownLatch countDownLatch = new CountDownLatch(1);
            try {
                watchNode(countDownLatch);
                countDownLatch.await();
                try {
                    doLock();
                } catch (Exception e) {
                    e.printStackTrace();
                    return false;
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                return false;
            }
            return true;
        }
    
        private void doLock() throws Exception {
            zkCli.createTempNode(LOCK_PATH, "123");
        }
    
        /**
         * 监听
         * @param countDownLatch
         */
        private void watchNode(CountDownLatch countDownLatch){
            try {
                if (zk.exists(LOCK_PATH,true) == null) {
                    System.err.println("当前锁节点不存在");
                    countDownLatch.countDown();
                }else{
                    zk.exists(LOCK_PATH, watchedEvent -> {
                        System.err.println("zkExit watch");
                        if (watchedEvent.getType() == Watcher.Event.EventType.NodeDeleted) {
                            System.err.println("delete Node");
                            countDownLatch.countDown();
                        }
                    });
                }
            } catch (KeeperException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    
        /**
         * 释放锁
         */
        public void unLock(){
            try {
                zkCli.deleteNode(LOCK_PATH);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (KeeperException e) {
                e.printStackTrace();
            }
        }
    
    ```

  * 共享锁

    **实现原理：**

    利用zookeeper创建临时有序节点特性，我们在获取锁的时候，在锁节点上创建一个有序节点，并记录创建返回后的节点`lockID`，再获取所有的节点信息，如果最小的节点信息是自己本身，那么久可以成功获取锁，否则的话就监听上一个比自己大的节点`lockIdLess`，如果节点`lockIdLess`被删除的话，则自己就可以成功获取锁。

    ![共享锁](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-zk2.jpg?raw=true)

    **代码实现：**

    基于java api 实现

    ~~~java
     private static final String LOCK_PATH = "/Locks/";
    
        private static ZkCliDemo zkCli;
    
        private static ZooKeeper zk;
    
        private String lockID;
    
        private CountDownLatch countDownLatch = new CountDownLatch(1);
        public LockTwo(){
            zkCli = new ZkCliDemo();
        }
    
        public void connect(){
            try {
                zk = zkCli.connectZookeeper();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    
        public boolean lock(){
            try {
                lockID = zkCli.createTempSeqNode(LOCK_PATH, "12");
                System.err.println(Thread.currentThread().getName()+ "-> 成功创建,lock节点[" +lockID+ "]"+"开始竞争锁");
                List<String> childNodes = zkCli.getChilds("/Locks");
                //排序
                SortedSet<String> sortedSet = new TreeSet<>();
                for (String str : childNodes) {
                    sortedSet.add(LOCK_PATH + str);
                }
                String first = sortedSet.first();
                if(lockID.equals(first)){
                    //表示成功获取锁
                    System.err.println(Thread.currentThread().getName()+ "-> 成功获取锁,lock节点[" +lockID+ "]");
                    return true;
                }
                //比当前节点更小的节点集合
                SortedSet<String> lessNodes = sortedSet.headSet(first);
                if (!lessNodes.isEmpty()) {
                    String preLockId = lessNodes.last();
                    zk.exists(preLockId,watchedEvent -> {
                        if (watchedEvent.getType() == Watcher.Event.EventType.NodeDeleted) {
                            countDownLatch.countDown();
                        }
                    });
                    countDownLatch.await(ZkCliDemo.SESSION_TIME_OUT, TimeUnit.MILLISECONDS);
                    System.err.println(Thread.currentThread().getName()+ "成功获取锁:[" +lockID+ "]");
                }
                return true;
            } catch (KeeperException e) {
                e.printStackTrace();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return false;
        }
    
        public void unlock(){
            System.err.println(Thread.currentThread().getName()+ "开始释放锁:[" +lockID+ "]");
            try {
                zkCli.deleteNode(lockID);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (KeeperException e) {
                e.printStackTrace();
            }
        }
    
        public static void main(String[] args) {
            final CountDownLatch countDownLatch = new CountDownLatch(10);
            for (int i = 0; i < 10; i++) {
                new Thread(() ->{
                    LockTwo lockTwo = new LockTwo();
                    lockTwo.connect();
                    countDownLatch.countDown();
                    try {
                        countDownLatch.await();
                        lockTwo.lock();
                        Thread.sleep(300);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }finally {
                        lockTwo.unlock();
                    }
                }).start();
            }
        }
    }
    ~~~

    

* 负载均衡

  使得客户端请求或者数据分摊多个计算机单元上。

* master 选举

  为了让服务高可用，也就是7*24小时可用， 99.999%可用。

  在集群中，存在Master - Slave 模式，那么怎么利用Zookeeper进行Master选举呢

  **实现原理**：Zookeeper不能重复创建一个节点的特性，每个客户端启动的时候，去zookeeper服务器上创建Master节点，创建成功则被选中为Master,否则就获取从zookeeper获取Master服务器信息，并且要监听这个节点，以便master挂掉后，重新进行选举。话不多说，直接看代码实现

  **代码实现**:zkClient的实现

  ~~~java
  /** 基于ZkClient 实现的Master选举过程
   * @Author tryingpfq
   * @Date 2020/3/28
   */
  public class MasterSelector {
      private ZkClient zkClient;
  
      /**
       * 注册节点内容变化
       */
      private IZkDataListener dataListener;
  
      private ServerInfo mySelf;
  
      private ServerInfo master;
  
      private static boolean isRunning = false;
  
      private ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
  
      public MasterSelector(ZkClient zkClient, ServerInfo serverInfo) {
          this.zkClient = zkClient;
          this.mySelf = serverInfo;
  
          this.dataListener = new IZkDataListener() {
              @Override
              public void handleDataChange(String s, Object o) throws Exception {
  
              }
  
              @Override
              public void handleDataDeleted(String s) throws Exception {
                  System.err.println("server:" + master.getServerId() + " has delete");
                  master = null;
                  //两秒之后重新进行选举
                  executorService.schedule(() ->{
                      chooserMaster();
                  },2,TimeUnit.SECONDS);
              }
          };
  
      }
  
      public void start(){
          if (!isRunning) {
              isRunning = true;
              System.out.println("server:"+ mySelf.getServerId()+" start");
              //注册节点事件
              zkClient.subscribeDataChanges(ZkCons.MASTER_PATH, dataListener);
              chooserMaster();
          }
  
      }
  
      private void chooserMaster(){
          if (!isRunning) {
              System.out.println("server:"+ mySelf.getServerId() + " is stop");
              return;
          }
          try {
              zkClient.createEphemeral(ZkCons.MASTER_PATH,mySelf);
              master = mySelf;
              System.out.println("server:" + master.getServerId() + "成功选举为master");
  
              //触发故障
              executorService.schedule(() -> {
                  releaseMaster();
              }, 5, TimeUnit.SECONDS);
          } catch (RuntimeException e) {
              ServerInfo info = zkClient.readData(ZkCons.MASTER_PATH);
              if (info == null) {
                  checkIsMaster();
              }else{
                  master = info;
              }
          }
  
      }
  
      public void stop(){
          if (isRunning) {
              isRunning = false;
              executorService.shutdownNow();
              zkClient.unsubscribeDataChanges(ZkCons.MASTER_PATH, dataListener);
              releaseMaster();
          }
      }
  
      /**
       * 模拟master故障
       */
      public void releaseMaster(){
          if (master == null) {
              return;
          }
          if (checkIsMaster()) {
              zkClient.delete(ZkCons.MASTER_PATH);
          }
      }
  
      public boolean checkIsMaster(){
          if (master == null || mySelf == null) {
              return false;
          }
          return mySelf.getServerId() == master.getServerId();
      }
  
      public class MasterChoosterTest {
  
      /**
       * 需要多模拟几个进程执行 就能看出master故障后重新选举的master
       * @param args
       * @throws InterruptedException
       */
      public static void main(String[] args) throws InterruptedException {
          ZkClient zkClient = new ZkClient(ZkCons.CONNECT_STR, ZkCons.SESSION_TIME_OUT, 			ZkCons.CONN_TIME_OUT);
  
          ServerInfo serverInfo = new ServerInfo(1, "try_" + 1);
  
          MasterSelector selector = new MasterSelector(zkClient, serverInfo);
          selector.start();
          try {
              TimeUnit.SECONDS.sleep(1);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
          Thread.sleep(50000);
      }
  
  ~~~

  基于Curator的实现：理由LeaderSelector就可以很方便的实现，**但要了解里面原理还是待学习**

  ~~~java
  package com.tryingpfq.fenbushi.zk.curator;
  
  import com.tryingpfq.fenbushi.zk.ZkCons;
  import org.apache.curator.framework.CuratorFramework;
  import org.apache.curator.framework.CuratorFrameworkFactory;
  import org.apache.curator.framework.CuratorTempFramework;
  import org.apache.curator.framework.recipes.leader.LeaderSelector;
  import org.apache.curator.framework.recipes.leader.LeaderSelectorListener;
  import org.apache.curator.framework.state.ConnectionState;
  import org.apache.curator.retry.ExponentialBackoffRetry;
  
  /**
   * 基于curator 工具的Master选举
   * @Author tryingpfq
   * @Date 2020/3/28
   */
  public class CuratorMasterSelector {
  
      public void chooser(){
          CuratorFramework curatorFramework = CuratorFrameworkFactory.builder().connectString(ZkCons.CONNECT_STR)
                  .retryPolicy(new ExponentialBackoffRetry(1000,3)).build();
  
          LeaderSelector leaderSelector = new LeaderSelector(curatorFramework, ZkCons.MASTER_PATH, new LeaderSelectorListener() {
              @Override
              public void takeLeadership(CuratorFramework curatorFramework) throws Exception {
                  System.err.println("获得leader 成功");
              }
  
              @Override
              public void stateChanged(CuratorFramework curatorFramework, ConnectionState connectionState) {
  
              }
          });
          leaderSelector.autoRequeue();
          //如果选举成功的话，会到上面takeLeadership中去
          leaderSelector.start();
      }
  }
  
  ~~~

  

* 分布式队列

  目前好像常用的还是消息中间件(ActiveMQ、kafka.......)，但用zk也是可以实现的。

  实现思路：先进先出队列

  1. 通过getChildren获取指定根节点下的所有子节点，子节点就是任务

  2. 确定自己节点在子节点中的顺序

  3. 如果自己不是最小的子节点，那么监控比自己小的上一个子节点，否则处于等待

     接收watcher通知，重复流程
     

* 

### 集群角色

​	上面已经提到过，zk中主要有三种角色，分别是：leader、follower、observer，而且集群中机器的是2n + 1台组成，为什么是奇数呢，是因为zookeeper的过半原则。

* leader:是zookeeper集群的核心。

  \1. 事务请求的唯一调度者和处理者，保证集群事务处理的顺序性

  \2. 集群内部各个服务器的调度者

* follower

  \1. 处理客户端非事务请求，以及转发事务请求给leader服务器

  \2. 参与事务请求提议（proposal）的投票（客户端的一个事务请求，需要半数服务器投票通过以后才能通知leader commit； leader会发起一个提案，要求follower投票）

  \3. 参与leader选举的投票

* observer

  观察zookeeper集群中最新状态的变化并将这些状态同步到observer服务器上，不参与事物请求和leader选举投票，所以增加observer不影响集群中事务处理能力，同时还能提升集群的非事务处理能力

### leader 选举

zk中有LeaderElection/AuthFastLeaderElection/FastLeaderElection三种选举算法。默认的是FastLeaderElection。下面大概分析一下过程，但具体实现还是要看源码

`org.apache.zookeeper.server.quorum.QuorumPeer#startLeaderElection`

serverid : 在配置server集群的时候，给定服务器的标识id（myid）

zxid : 服务器在运行时产生的数据ID， zxid的值越大，表示数据越新

Epoch: 选举的轮数

server的状态：Looking、 Following、Observering、Leading

![选举流程](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-zk3.jpg?raw=true)



### ZAB协议

ZAB(zab Zookeeper atomic broadcast)协议，基于paxos协议的一个改进，并没有完全采用paxos算法。paxos协议主要就是如何保证在分布式环网络环境下，各个服务器如何达成一致最终保证数据的一致性问题。

zab协议为分布式协调服务zookeeper专门设计的一种支持`崩溃恢复`的`原子广播`协议

#### zab协议的原理

1. 在zookeeper的主备模式下，通过zab协议来保证集群中各个副本数据的一致性

2. zookeeper使用的是单一的主进程来接收并处理所有的事务请求，并采用zab协议，把数据的状态变更以事务请求的形式广播到其他的节点

3. zab协议在主备模型架构中，保证了同一时刻只能有一个主进程来广播服务器的状态变更

4. 所有的事务请求必须由全局唯一的服务器来协调处理，这个的服务器叫leader，其他的叫follower。

​	leader节点主要负责把客户端的事务请求转化成一个事务提议（proposal），并分发给集群中的所follower节点再等待所有follower节点的反馈。一旦超过半数服务器进行了正确的反馈，那么leader就会commit这条消息

#### zab协议的工作原理

​	**1:什么情况下zab协议会进入崩溃恢复模式**

​		i. 当服务器启动时

​		ii. 当leader服务器出现网络中断、崩溃或者重启的情况

​		iii. 集群中已经不存在过半的服务器与该leader保持正常通信

​	**2:zab协议进入崩溃恢复模式会做什么**

​		i. 当leader出现问题，zab协议进入崩溃恢复模式，并且选举出新的leader。当新的leader选举出来以后，如果集群中已经有过半机器完成了leader服务器的状态同（数据同步），退出崩溃恢复，进入消息广播模式

​		ii. 当新的机器加入到集群中的时候，如果已经存在leader服务器，那么新加入的服务器就会自觉进入数据恢复模式，找到leader进行数据同步

![zab](https://github.com/tryingpfq/tryingpfq.github.io/blob/master/picture/bg-zk4.jpg?raw=true)

​	

* 假设一个事务在leader服务器被提交了，并且已经有过半的follower返回了ack。 在leader节点把commit消息发送给folower机器之前leader服务器挂了怎么办？

   	zab协议，一定需要保证已经被leader提交的事务也能够被所有follower提交

  ​	 zab协议需要保证，在崩溃恢复过程中跳过哪些已经被丢弃的事务



### 源码分析
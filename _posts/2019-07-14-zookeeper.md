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

### 中性化和去中性化
假如在分布式系统中，所有节点都围绕一个中心节点去通信或者数据同步，那么假如当这个节点挂掉的时候，这个分布式系统就蹦了。所以在分布式中，每一个节点都是高度自治的，节点之间可以自由连接，任何一个节点都可能成为阶段性的中心，但不具备强制性的中性控制能力。分布式架构里面，很多的架构思想采用的是：当集群发生故障的时候，集群中的人群会自动“选举”出一个新的领导。
最典型的是： zookeeper / etcd


### CAP/BASE理论
**CAP：在分布式中，不管如何，同时只能满足两种条件。CP or AP** 

 * C（一致性 Consistency）: 所有节点上的数据，时刻保持一致
 * A可用性（Availability）：每个请求都能够收到一个响应，无论响应成功或者失败
 * 分区容错 （Partition-tolerance）：表示系统出现脑裂以后，可能导致某些server与集群中的其他机器失去联系

CAP理论仅适用于原子读写的Nosql场景，不适用于数据库系统

**BASE**

基于CAP理论，CAP理论并不适用于数据库事务（因为更新一些错误的数据而导致数据出现紊乱，无论什么样的数据库高可用方案都是
徒劳），虽然XA事务可以保证数据库在分布式系统下的ACID特性，但是会带来性能方面的影响；
eBay尝试了一种完全不同的套路，放宽了对事务ACID的要求。提出了BASE理论
Basically available ： 数据库采用分片模式， 把100W的用户数据分布在5个实例上。如果破坏了其中一个实例，仍然可以保证
80%的用户可用

soft-state：在基于client-server模式的系统中，server端是否有状态，决定了系统是否具备良好的水平扩展、负载均衡、故障恢复等特性。
Server端承诺会维护client端状态数据，这个状态仅仅维持一小段时间, 这段时间以后，server端就会丢弃这个状态，恢复正常状态

Eventually consistent：数据的最终一致性



### ZAP协议




### 基本概念
zookeeper是一个典型的分布式协调框架，具有分布式数据一致性的解决方案。它主要用在数据的发布/订阅（配置中心:disconf）、 负载均衡（dubbo利用了zookeeper机制实现负载均衡） 、命名服务、master选举(kafka、hadoop、hbase)、分布式队列、分布式锁。zookeeper的特性，数据一致性：从同一个客户端发起的事物请求，最终会严格按照顺序应用到zookeeper中，原子性：所有的事务请求的处理结果在整个集群中的所有机器上的应用情况是一致的，也就是说，要么整个集群中的所有机器都成功应用了某一事务、要么全都不应用，可靠性：一旦服务器成功应用了某一个事务数据，并且对客户端做了响应，那么这个数据在整个集群中一定是同步并且保留下来的，实时性：一旦一个事务被成功应用，客户端就能够立即从服务器端读取到事务变更后的最新数据状态；（zookeeper仅仅保证在一定时间内，近实时）。


### zookeeper的安装
[下载zookeeper](http://apache.fayea.com/zookeeper/zookeeper-3.5.5/) 目前版本是3.5.5,下面是基于Linux的安装，可以使用虚拟机，也方便进行集群搭建。

[下载](http://mirrors.hust.edu.cn/apache/zookeeper/stable/)

#### 单机搭建

* 1.	解压zookeeper tar -zxvf zookeeper-3.4.10.tar.gz
* 2.	cd到 ZK_HOME/conf  , copy一份zoo.cfg
* 3.    cp  zoo_sample.cfg  zoo.cfg
* 4. sh zkServer.sh xxx 查看命令 {start|start-foreground|stop|restart|status|upgrade|print-cmd}

     启动命令 sh zkServer.sh start
* 5.	sh zkCli.sh -server  ip:port

#### 集群搭建

1：修改zoo.cfg
	128/129/130
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



### 概念

#### 数据模型

zookeeper的数据模型和文件系统类似，每一个节点称为：znode. 是zookeeper中的最小数据单元。每一个znode上都可以

保存数据和挂载子节点。 从而构成一个层次化的属性结构

**节点特性**

一般在分布式语境下的节点是指组成集群的每一台服务器，在 zookeeper 中还有另外一层意思，称之为数据节点（ZNode）。
 zookeeper 的整个名字空间的结构是层次化的，和 Linux 文件系统结构相似，是一颗树。名字空间的层次由斜杠（/）来进行分割，在名称空间里面的每一个节点的名字空间由这个结点的路径来确定。
 每个 ZNode 上都会保存自己的数据内容，同时还会保存一系列属性信息。

持久化节点 ： 节点创建后会一直存在zookeeper服务器上，直到主动删除

持久化有序节点 ：每个节点都会为它的一级子节点维护一个顺序

临时节点 ： 临时节点的生命周期和客户端的**会话**保持一致。当客户端会话失效，该节点自动清理

临时有序节点 ： 在临时节点上多勒一个顺序性特性

**Watcher**

**zookeeper**提供了分布式数据发布/订阅,zookeeper允许客户端向服务器注册一个watcher监听。当服务器端的节点触发指定事件的时候会触发watcher。服务端会向客户端发送一个事件通知watcher的通知是一次性，一旦触发一次通知后，该watcher就失效。

**ACL**

zookeeper提供控制节点访问权限的功能，用于有效的保证zookeeper中数据的安全性。避免误操作而导致系统出现重大事故。

CREATE /READ/WRITE/DELETE/ADMIN

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



### 应用

* 统一命名服务
* 配置管理
* 分布式锁
* 
* 



 
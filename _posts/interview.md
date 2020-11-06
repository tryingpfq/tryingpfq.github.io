## 面试



### Java基础

1：谈谈你对Java平台的理解，“Java是解释执行”，这句话正确吗？

java为什么能跨平台，称为一次编译，到处运行。就是因为源代码编译后，是一个class文件，也就是字节码，这个字节码是需要解释器来进行解释为机器码(可以直接在机器上运行的),因为对于不同的平台，有不同的JVM，所以才让java是跨平台的。

当然java是解释执行的，总的来说，是对的，但也有例外，对于代码也有28原则，有些热点代码，会被JIT动态编译器直接编译成机器码，这个是缓存在codeCache中的。



参考：极客时间：Java核心技术面试精讲



2： Exception 和 Error有什么区别，运行时异常与一般异常有什么区别, NoClassDefFoundError 和 ClassNotFoundException 有什么区别

检查异常：这种异常现实中难以避免，比如sleep,文件操作，必须处理（try....catch）,编译器会给你检查，如果没处理，就无法编译。

非检查异常：这种异常是自己可以避免的，比如数组越界，NPE,

参考：极客时间：Java核心技术面试精讲



3：谈谈final、finally、finalize有什么不同？

final:可以用来修饰类，方法，变量，分别有不同的含义，修饰class代码这个类不能被继承和扩展，修饰变量代表这个变量不可以被修改，而final的方法也是不可以被重写的。

finallly则是Java保证重点代码一定要被执行的有一种机制，比如JDBC连接，锁的unlock。

finalize是基础类java.lang.Object的一个方法，它的设计目的是保证对象在被垃圾收集前完成特定资源的回收。finalize机制现在已经不推荐使用，并且在JDK 9开始被标记为deprecated。



4：强引用、弱引用、软引用、虚引用、幻引用有什么区别

[参考](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=356#/detail/pc?id=4194)

参考：极客时间：Java核心技术面试精讲



5：String、StringBuffer、StringBuilder有什么区别？

首先，String是一个不可变的类，里面是一个char数组，所以在每次用到string（拼接、裁剪）的时候，都需要创建新的String对象。

由于字符串操作的普遍性，所以相关操作的效率往往对应用性能哟明显的影响。

StringBuffer是为了解决上面提到拼接产生太多中间对象的问题提供的一个类，提供了append和add方法，其实就是char数组的一个拷贝。这个是线程安全的，另外一个非线程安全的就是StringBuilder

参考：极客时间：Java核心技术面试精讲



6：动态代理基于什么原理

反射



7：int和Integer有什么区别



8：对比Vector、ArrrayList、LinkedList有何区别

老生常谈的问题吧，这三个都是集合框架中的List，Vector和Array底层数据结果是数组实现，区别是Vector是线程安全的，而ArrayList是非线程安全的，当然对于线程安全的集合来说，也可以通过Collections工具中的syn---方法来实现。数组，显然是一般对于插入来说是比较低效的，因为涉及到数据的迁移，但如果都是尾插的话，倒不影响，查询效率会高一下，可以随便访问。然后LinkedList底层是一个双向链表，也是非线程安全的，对于插入和删除来说，会比较高效一点。



9：对比Hashtable、HashMap、TreeMap有什么不同



10：如何保证集合是线程安全的，ConcurrentHashMap如何实现高效地线程安全？



11：Java提供了哪些IO方式？NIO如何实现多路复用？

传统的java.io包，是基于流模型实现的，提供最熟知的一些IO功能，比如File抽象，输入输出等，交互方式是同步、阻塞的方式，也就是说，在读取输入流或者写入输出流时，在读、写动作完成之前，线程会一直阻塞在那里，它们之间的调用是可靠的线性顺序。

也把java.net下面提供的部分网络API，比如Socket、ServerSocket、HttpURLConnection也归类到同步阻塞IO类库，因为网络通信同样是IO行为。

在Java1.4中引入了NIO框架，提供了Channel.Selector.Buffer等抽象，可以构建多路复用。同步非阻塞IO.

在Java7中，引入了AIO,
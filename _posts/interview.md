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




《Effective Java》



### 创建和销毁对象

这里的主题是创建和销毁对象：何时以及如何创建对象，何时以及如何避免创建对象，如何确保他们能够在合适的时间销毁，以及如何管理对象销毁之前必须进行的各种清理操作。

`1：用静态工厂代替构造器。`

平时，我们习惯通过new然后去创建对象，那是否可以考虑提供一个共有的静态工厂方法，只是用来返回类的实类静态方法。



2：`遇到多个构造器参数时要考虑使用构造器`

静态工厂和构造器有个共同的局限性：他们都不能扩展到大量的可选参数。经常程序员可能习惯用重叠构造器模式，可是当遇到很多参数的时候，可能就不太方便了，也不方便阅读。

然后可能会想到一个方法。就是下面这样。

```java
vo.setA(a);
vo.setB(b);
vo.setC(c);
.....
你能知道这样会有什么问题吗，不仅仅是可能要写很多行的set方法。
这样vo可能处于不一致的桩体，因为如果是在构造函数中是并发安全的(这个具体，待研究)
   
```

平时有没有遇到过通过建造者模式来处理这种多参数的构造，而且还可以对一些必须的参数进行校验，如果检查失败，就可以抛出IllegalArgumentException。这样的代码是不是很容易编写，而且易于阅读。

与构造器相比，builder的稍微优势在于，它可以有多个可变参数，因为builder是利用单独的方法来设置每一个参数。

而且Builder模式比较灵活，可以自动填充需要某些域。但也存在一些不足，为了创建对象，必须先创建它的构造器。简而言之，如果类的构造器或者静态工厂中具有多个参数，通过Builder模式是一个不错的选择。



3：`用私有构造器或者枚举类型强化Singleton属性`

这里不多说，但要注意几个点

可以借助AccessibleObject.setAccessible()方法，通过反射机制调用私有构造器。

若实现了Serializable接口的单列，为了维护并保证Singleton，必须申明所有实例域都是瞬间的，并提供一个readResolve方法。否则每次反序列化都会破坏单例。

但如果是通过枚举就不同了，无偿的提供了反序列化机制。单元素的枚举类型经常被成为实现单例的最佳实战。



4：`通过私有构造器强化不可实例化的能力`

假如一些工具类，提供的是一些静态方法，不需要被实例化，可以把构造私有化。



5：`优先考虑依赖注入来引用资源`

这里很简单，其实就是通过构造传入进来就好。



6：`避免创建不必要的对象`

这个怎么说呢，其实就是就是有些对象可以重用的话，可以缓存起来，避免重复创建。例如，在Map中，当调用keySet方法获取keys的时候，每次都是返回同一个set对象的，只是里面的值可能会有修改。

看段代码

```java
private static long sum(){
    Long sum = 0L;
    for(long i = 0; i < Integer>MAX_VALUE; i++){
        sum += i;
    }
    return sum;
}
//这段代码会有什么问题呢
就因为把sum定义为Long,导致加的时候要自动装箱(这里就会创建Long对象)。所以要优先考虑使用基本类型 而不是装箱基本类型，要当心无意识的自动装箱。
    
```

7：`消除过期的对象引用`

可能，对于Java程序员来说，大部分人会觉得不需要考虑内存管理的事情，JVM会帮我们做，其实不然。

继续看段代码

```java
package com.tryingpfq.effectiveJava;

import java.util.Arrays;
import java.util.EmptyStackException;

/**
 * @author tryingpfq
 * @date 2020/7/7
 **/
public class Stack<T> {

    private Object[] elements;

    private int size;

    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack(T t){
        elements = new  Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(T obj) {
        ensureCapacity();
        elements[size++] = obj;
    }

    public T pop(){
        if (size == 0) {
            throw new EmptyStackException();
        }
        return (T) elements[--size];
    }
    private void ensureCapacity(){
        if(elements.length == size){
            elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }

}

```

看上去没什么问题，但其实存在一个内存泄漏的问题。下面分析一下哈，

如果这个栈中的元素先是增长，然后再收缩，那么从栈中弹出来的对象将不会被当做垃圾回收，即使使用栈的程序不再使用这些对象，他们也不会被回收。因为栈内部维护这对这些对象的过期引用。相比说到这里应该知道了吧。修改也很简单的，修改下出栈就可以了。

```java
    public T pop(){
        if (size == 0) {
            throw new EmptyStackException();
        }
        T ele = (T) elements[--size];
        elements[size] = null;
        return ele;
    }
```

但要注意的是，清空对象引用应该是一种例外，而不是一种规范行为。



8：`避免使用终结方法和清楚方法`



9：`try -with-resource 优先于 try-finally`

经常我们在需要关闭资源的时候，会习惯用到try-finally，即使抛异常了也可以正常关闭，如果是只关闭单个的话，看起来还好，如果在需要多个的情况下，可能看起来就比较乱。[参考博客](https://juejin.im/entry/57f73e81bf22ec00647dacd0)



### 对于所有的对象都通用

尽管Object是具体一个类，但设计它主要还是为了扩展，



10：`覆盖equals时请遵循通用约定`

覆盖equals方法看起来很简单，但是有许多覆盖方式会导致错误，并且后果可能非常严重。

* 类的每个实例本质上都是唯一的

  对于代表活动实体而不是值的类来说确实如此，例如Thread,Object

* 超类已经覆盖了equals，超类的行为对于这个类也是合适的

  比如大多数的Set实现都从AbstractSet继承equals方法，List从AbstractList继承，Map实现AbstractMap继承。

* 类是私有的，或者包级私有的，可以确定它的equals方法永远不会被调用。



在覆盖equals方法的时候，必须遵守它的通用约定，下面是一些约定的内容。

* 自反性：这个还是好理解的，就是对于任何非null的引用x,x.equals(x) 为true
* 对称性
* 传递性
* 一致性 
* 对于任何非null的引用值x，x.equals(null) 必须返回false



11：`覆盖equals时总要覆盖hashCode`

在覆盖了equals的时候，都必须覆盖hashCode方法，如果不这样做的话，就会违反hashCode的通用约定，从而导致该类无法结合所有基于散列的集合一起正常运作，这类集合包括HashMap 和 HashSet。

* 如果两个对象根据equals(Object) 方法比较是相等的，那么调用这两个对象中的hashCode方法都必须产生相同的整数结果。



12：`始终要覆盖toString`



13：`谨慎的覆盖clone`

[博客](https://blog.csdn.net/zhangjg_blog/article/details/18369201)



14：`考虑实现Comparable接口`



### 类和接口

类和接口是Java编程语言的核心，它们也是Java语言的基本抽象单元。

15：`使类和成员的可访问性最小化`



16：`要在公有而非共有域中使用访问方法`



17：`使可变性最小化`



18：`复合优先于继承`



19：`要么设计继承并提供文档说明，要么禁止继承`



20：`接口优于抽象类`



21：`为后代设计接口`



22：`接口只用于定义类型`



23：`类层次优于标签类`



24：`静态成员类优先于非静态成员类`



25：`限制源文件为单个顶级类`

永远不要把多个顶级类或者接口放在一个源文件中，遵循这个规则可以确保编译时一个类不会有多个定义。



### 泛型

泛型，我觉得最常见的应该就是集合了，如果没有泛型，从集合中读取到的每一个对象都必须进行转换，如果不小心插入了类型错误的对象，编译的时候并不会报错，但在运行的时候，就会出现类型转换错误。



26：请不要使用原生态类型


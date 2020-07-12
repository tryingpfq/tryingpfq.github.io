



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



26：`请不要使用原生态类型`

什么是原生类型呢，每一种泛型都定义一种原生态类型，即不带任何实际类型参数的泛型名称。例如，与List<E>相对应的原生态类型是List。原生态类型就像从类型声明中删除了所有的泛型信息。

也就是不要这样用

```java
private final Collection stamps = .....
```



如果使用原生态类型，就失去了泛型在安全性和描述性方面的所有优势。

既然不要使用原生态类型，那么为什么java的设计者还是运行这样做呢，大概是为了提供兼容性，早起是没有出现泛型的。



27：`消除非受检的警告`

用泛型编程时，经常会遇到一些警告。当遇到一些警告时，要即的去消除。

比如

```java
Set<Integer> sets = new HashSet();
//这了就会出现警告
怎么修改呢，只需要加个<>就OK了，并不需要制定真正的类型。
```



如果无法消除警告，同时可以证明引起警告的代码类型是安全的，在这种情况下，可以使用@SuppressWarnings("unchecked")注解来禁止这条警告。但这个注解的使用，还是要注意点，尽量粒度小一点，小在一个字段上，千万不要在这个类上使用。



28：`列表优于数组`

来看一种情况

```java
Object[] array = new Long[1];
array[0] = "i do";	//很明显这个是错误的

List<Object> list = new ArrayList<Long>();
list.add("i do");//明显也是不对的

//出现这种问题，我们肯定是希望在编译的时候就发现，但如果是数组的话，要在运行的时候才会发现。
```

总之，数组是协变且可以具体化的，而泛型是不可变的且可以被擦除的。



29：`优先考虑泛型`

一般，直接用一些集合声明参数化，或者JDK中已经提供了的泛型方法，这些都不难，但编写自己的泛型就比较难了。

继续以上面提到过的Stack为类。里面的的值就应该被参数化，即可以考虑泛型化。

```java
public class Stack<E> {

    //1
    private E[] elements;

    private int size;

    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack(E t){
        //2
        elements = new  E[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E obj) {
        ensureCapacity();
        elements[size++] = obj;
    }

    public E pop(){
        if (size == 0) {
            throw new EmptyStackException();
        }
        return (e) elements[--size];
    }
    private void ensureCapacity(){
        if(elements.length == size){
            elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }

}


```

但是看到上面代码1 2地方，你会发现有警告或者错误提示，也就是说不可创建不可具体化的类型数组，如E，每当编写用数组支持的泛型时，都会出现这种问题，那怎么解决呢

* 直接绕过创建泛型数组指令，创建一个Object数组，然后强转为E数组，但也会有警告。（这里只需要强转一次）
* E[]数组替换我Object[]数组，pop的时候就要强转一下(每次都需要强转).



简之，使用泛型比使用需要在客户端代码中进行强转的类型来的更安全一些，所以通常要把类做成泛型的。



30：`优先考虑泛型方法`

正如类可以从泛型中受益一般，方法也一样，特别是对于静态工具方法更适合于泛型化。

以合并两个集合为类

```java
public static <E> Set<E> union(Set<E> s1,Set<E> s2){
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```



31：`利用有限制通配符来提升API的灵活性`

首先要知道，什么是有限制通配符

```java
public void pushAll(Iterable<? extends E>){
    
}

public void popAll(Colelction<? super E> des){
    
}
```

其实上面可以这样理解的，如果参数化类型，表示一个生产者T，就使用<? extends T>，如果他表示一个消费者，就使用<? supper T>

那么上面的代码是否可以考虑修改一下呢

```java
public static <E> Set<E> union(Set<? extends E> s1,Set<? extends E> s2){
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```



32：`谨慎并用发型和可变参数`

一旦要用，就需要使用注解@SafeVarargs

```java
@SafeVarargs
static<T> List<T> flatten(List<? extends T>... lists){
    List<T> result = new ArrayList<>();
    for(List<? extends T> list : lists){
        result.addAll(list);
    }
    return result;
}

//如果不想用上面注解，那怎么解决呢
static<T> List<T> flatten(List<List<? extends T>>... lists){
    List<T> result = new ArrayList<>();
    for(List<? extends T> list : lists){
        result.addAll(list);
    }
    return result;
}
```



33：`优先考虑类型安全的异构容器`



### 枚举和注解



34：`用enum代替int常量`



35：`用实例域代替序数`

这个怎么说呢，对于所有的枚举，都要一个ordinal方法，它返回的是每个枚举常量在类型中的数字位置。一旦枚举常量前后顺序互换，那么久会出现问题，特别是对于一int持久化了的，那么反序列化后就不对了。所以永远不要根据枚举的序数导出与它关联的值，而是在它保存在一个实例域中。

其实ordinal这个方法，大多数程序员是不需要用到的，这个是方法EnumSet和EnumMap这种基于枚举的通用数据结构。



36：`用EnumSet代替位域`



37：`用EnumMap代替序数索引`



38：`用接口模拟可扩展的枚举`

对于枚举来说，是不可扩展的，但可以通过编写接口以及实现该接口基础枚举类型类对它进行模拟。



39：`注解优于命名模式`



40：`坚持使用Override注解`



41：`用标记接口定义类型`





### Lambda 和 Stream

在java8中，增加了函数接口、Lambda和方法引用，使得创建函数对象变得容易。



42：`Lambda优先于匿名类`

创建函数对象的主要方式是通过匿名类，看下这段代码，通过字符串长度排序。

```java
Collections.sort(words,new Comparator<String>(){
    public int compare(String s1,String s2){
        return Integer.compare(s1.length(),s2.length)
    }
});
```

上面的代码中，就是通过创建了一个匿名类。看起来是比较繁琐的，如果用lambda表达式，那么代码就会简单很多。

```java
Collections.sort(words,(s1,s2) -> Integer.comare(s1.length(),s2.length()));
```

编译器会通过函数推到，判断参数类型这些。

```java
//上面代码其实还可以更简单一些
Collections.sort(words,comparingInt(Sring::length));

words.sort(comparintInt(String:length));
```



然而并不是所有这些工作都可以用lambda来完成，毕竟lambda限于函数接口，如果是抽象类的实例，那么就应该考虑匿名类来完成。或者存在多个抽象方法的接口，也应该考虑匿名类。



43：`方法引用优先于Lambda`

与匿名类相比，Lambda的主要优势在于更加简洁，java还提供了生成比Lambda更简洁函数对象的方法：方法引用。



44：`坚持使用标准的函数的接口`



45：`谨慎使用Stream`



46：`优先选择Stream中无副作用的函数`



47：`Stream要优先用Collection作为返回类型`



48：`谨慎使用Stream并行`


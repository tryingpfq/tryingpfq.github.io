读书笔记



### 《重构改善既有代码的设计》



方法：方法不宜过长，一个方法，一般只做一件事。

参数：如果会对方法参数有修改，要注意，尽量看能否用作返回值接收。



#### 重构原则

##### 为何重构



重构：重构是使用一系列重构手法，在不改变软件可观察行为的前提下，调整其结构。



可读性

要注意，自己写的代码，不仅仅是在计算机上能够正常运行就好，还要提高可读性，毕竟你的代码，可能会被未来一个开放者(或许很有可能就是自己)，如果另一个同事需要在你的代码上做修改，那么他必修要理解，所以自己代码写的话一定要让别人方便阅读和理解。



注意点

相同的功能代码一定不要在多个地方出现，比如：某个功能代码，多个地方有，然后也有被调用，当要对这个功能进行修改的时候，就要修改所有的地方，否则就出现问题，所以在一个地方实现就好。(我刚改的一个地方，就是玩家登陆初始化英雄和游戏中创建英雄再初始化的中的初始化必须只能有一个地方实现，如果分开的话就可能出现不一致问题)。



##### 何时重构

1：添加功能时重构

2：修补错误时重构

3：复审代码时重构

不过，平时在开发中，就要多注意代码质量。



##### 重构难题

1：接口的重构

当你需要修改函数名称时候，尽量保留旧函数，让它调用新函数，如果是就简单的复制函数，那么可能让你陷入重复代码的泥淖中无法自拔。(上次我在修改一个方法的时候，就犯了这种错误，不过不是接口方法)。

当然有时候需要对方法参数做修改的话，就不得不重新设计这个接口了





#### 代码的坏味道

1: Duplicated Code (重复代码)



2: Long Method

我们知道，在平时中，尽量方法不要过长，不然很难被理解，那么该如何避免过长呢，就需要进行提炼，也就是讲功能性代码提取成一个方法。

如何确定该提炼哪一段代码呢?一个很好的技巧是：寻找注释。它们通常能指出代码用途和实现手法之间的语义距离。如果代码前方有一行注释，就是在提醒你：可以将这段代码替换成一个函数，而且可以在注释的基础上给这个函数命名。就算只有一行代码，如果它需要以注释来说明，那也值得将它提炼到独立函数去。

条件表达式和循环常常也是提炼的信号。



3: Large Class 



4: Long Parameter List (过长参数列表)



·········



#### 重构列表

每一种重构手法都有五个部分

名称：

概要：简单介绍此一重构手法的适用情景，以及它所做的事情。

动机：为你介绍“为什么需要这个重构”和“什么情况下不该使用这个重构”。

做法：简明扼要地一步一步介绍如何进行此一重构。

范例：以一个十分简单的例子说明此重构手法如何运作。



#### 重新组织函数

1：`Extract Method 函数提炼`



2：`Inline Method 内联函数`



3：`Inline Temp 内联临时变量`

```java
int price = order.basePrice();
return price > 1000;

//重构后
return order.basePrice() > 1000;
```





4：`Replace Temp With Query 以查询取代临时变量`

```java
int price = _quality * ratio;
if(price > 1000){
    return true;
}
return false;

//重构后
if(price() > 1000){
    return true;
}
return false;

```



5：`Introduce Explaining Variable 引入解释性变量`

 这种怎么说呢，就是加入存在复杂的表达式 || && ，这中情况下就可以引入临时的解释性变量，但我觉得用处不大。



6：`Split Temporary Variable 分解临时变量`

加入你的程序有某个临时变量被赋值超过一次，并且它既不是循环变量，也不被用于收集结果，针对每次赋值，创造一个独立、对于的临时变量。

```java
double temp = 2 * (_height + _width);
System.out.println(temp);
temp = _heigh * _width;
System.out.println(temp);
// 其实这种情况 我觉得在以前开发中，自己还是这样写过这样的代码，就是为了不再定义新的变量。
// 觉得吧，如果这个变量表达的含义一样，还是可以不该的。

// 应该这样修改
final double perimeter = 2 * (_height + _width);
System.out.println(perimeter);
final double area = _height * _width;
System.out.println(area);
```



7：`移除对参数的赋值`

代码对一个参数进行赋值，以一个临时变量取代该参数的位置。

```java
int discount(int inputValue,int quality,int yearToDate){
    if(inputValue > 50){
        inputValue -= 20;
    }
}

// 加上关键字final 从而强制它遵循“不对参数赋值”这一惯例
int discount(final int inputValue,int quality,int yearToDate){
   	int result = inputValue;
    
    if(result > 50){
        result -= 20;
    }
}
```

这里其实要注意的就是，不要随便对参数赋值。这个地方要注意点点按值传递和按引用传递。



8：`Replace  Method With Method Object 以函数对象取代函数`

怎么说呢这个，其实就是加入某个方法太复杂，局部变量太多，就可以专门抽取一个类，在这个类中对这个方法就进行拆解。



9：`Substitute Algorithm 替换算法`

假如你想要把一个算法替换成一个更清晰的算法，将函数本地替换为另一个算法。

```java
String foundPerson(String[] people){
    for(int i  = 0 ; i < people.length ; i++){
        if(people[i].equal("peng")){
            return "peng";
        }
        if(people[i].equal("Bob")){
            return "Bob";
        }
    }
    return "";
}

//替换后
String foundPerson(String[] people){
    List<String> candidates = Arrays.asList(new String[]{"peng","Bob"});
    for(int i = 0 ; i < people.length ; i++){
        if(candidates.contains(people[i])){
            return people[i];
        }
    }
    return "";
}
```



#### 在对象之间搬移特性

1：`Move Method 搬移函数`



2：`Move Filed 搬移字段`

在你的程序中，某个字段被其所驻类之外的另一个类更多的用到，在目标类中新建一个字段，修改原字段所有的用户，令他们改用新字段。

这个意思其实很简单，就是对于一个类中的字段，如果这个字段在另一个类中用的还更多，那么就可以把这个地段移到另一个字段中去，之前有对原始字段有引用的地方，都进行修改。



3：`Extract Class 提炼类`

假如某个类做了应该由两个类做的事，那么就应该建立一个新类，将相关的字段和函数从旧类搬移到新类当中。



4：`Inline Class 将类内联化`

假如某个类没有做太多事情，将这类所有特性搬移到另一个类中，然后移除原类。



5：`Hide Delegate 隐藏委托关系`

这里直接看个案例，假如有两个类Person 和 Department结构如下

```java
class Person{
    Department department;
    
    public Department getDepartment(){
        return departement;
    }
    
    public void setDepartment(Departement department){
        this.department = department;
    }
}

class Department {
    private String chargeCode;
    private Person manager;
    
    public Department(Person person){
        this.manager = person;
    }
    
    public Person getManager(){
        return manager;
    }
}


```

//如果客户希望知道某人是经理的话，就必须要获得departement对象
manager = peng.getDepartment().getManager();

这样的编码就是对客户揭露了Department的工作原理，于是客户知道：Depa-rtment用以追踪“经理”这条信息。如果对客户隐藏Department，可以减少耦合。



那对于这个问题该如何解决呢，可不可考虑在Person类中做一个获取Department的委托函数

```java
public Person getManager(){
    return department.getManager();
}
```

其实这个在平时还是比较常见的，比如一些玩家模块，在获取模块中数据的时候，我们会在player中做一个委托函数，特别是对于PlayerEnt中的数据。



6：`Remove Middle Man 移除中间人`

某个类做了过多的简单委托动作，让客户直接调用受委托类。这个怎么说呢，其实就是对于上面那点重构，如果有大量函数需要这么做，那么就不得不在person之中安置大量委托行为。这个时候该怎么做呢，这就要移除中间人，也就是在Person中建立一个函数用于获得委托对象。



7：`Introduce Foregin Method 引入外加函数`

**没怎么理解**



8：`Introduce Local Extension 引入本地扩展`

这个问题怎么说呢，也就是在不能修改原始类源码，但又需要添加一些额外的方法，那么久可以自己扩展一个类，这个类作为原始类的子类。



#### 重新组织数据

1：`Self Encapsulate Field 自封装字段`

这点怎么说呢，其实就是在进行“public字段访问”的时候，到底是直接访问字段名，还是通过一个方法来获取。比如想访问超类的一个字段，一般是通过方法区访问可能会好些。

```java
private int _low,_high;
boolean includes(int arg){
    return arg >= _low && arg <= _high;
}

//重构为
boolean includes(int arg){
    return arg >= getLow() && arg <= getHigh();
}
```



2：`Replace Data Value With Object 以对象取代数据值`

就是你有一个数据项，需要与其他数据和行为一起使用才有意义。将数据项变成对象。还是直接看例子吧

```java
//订单类 需要用到customer
class Order{
    public Order(String customer){
        this.customer = customer;
    }
}

class Customer {

	public Customer(String name){

		_name = name;

	}

	private final String _name;

}


//如果在Order中 不是一个String 而是一个customer对象 会不会更好扩展呢
```



3：`Change Value to Reference 将值对象改为引用对象`



4：`Change Reference to Value 将引用对象改为值对象`



5：`Replace Array With Object 以对象取代数组`

假如有这样一个数组，就是每个元素代表不同的含义。那么，这个就可以换成一个对象，然后每个元素就是一个字段。

```java
String[] row = new String[3];
row[0] = "Liverpool";
row[1] = "15";

//向上面的这种就可以该成
Performance row = new Performance();
row.setName("Liverpool");
row.setWins("15");

//其实在开发中，可能会用到唯一的一对K V的 Pair
```


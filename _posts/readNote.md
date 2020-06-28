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



6：`Duplicate Observed Data 复制“被监视数据”`

其实就是将发送给客户端的数据和本身业务上的数据区分开。



7：`Change Unidirectional Association to Bidirectional 将单向关联改为双向关联`

就是假如在两个实例中，需要使用到对方的特征，也就是说就需要互相引用。还是直接看代码吧。

比如有两个类，Order 和 Customer,开始的时候，order中引用了customer，但customer中没有引用order。

```java
class Order{
    private Customer _customer;
    
    public Customer getCustomer(){
        return _customer;
    }
    
    public void setCustomer(Customer customer){
        this._customer = customer;
    }
}

class Customer{
    
}
```

显然，在业务中，一个customer可能存在多个order的，那么这就需要通过order来进行关联关系。

```java
class Order{
    void addCustomer(Customer customer){
        customer.frientOrders.add(customer);
        _customers.add(customer);
    }
    
    void removeCustomer(Customer customer){
        customer.friendOrders.remove(customer);
        _customers.remove(customer);
    }
}

class Customer{
    void addOrder(Order order){
        order.addCustomer(this);
    }
    
    void removeOrder(Order order){
        order.removeCustomer(order);
    }
}
```



8：`Change Bidirectional Association to Unidirectional 将双向关联改为单向关联`

假如存在两个类之前是双向关联的，但现在一个类不再引用另一个类，那么就需要该为单向关联。



9：`Replace Magic Number With Symbolic Constant 以字面常量取代魔法数`

假如你有一个字面数值，带有特殊含义。那么可以创建一个常量，根据其意义为它命名， 并将上面的字面数值替换为这个常量。

其实这个中情况平时还是比较常见的，一般都会专门定义为常量。

```java
double potentialEnergy(double mass,double height){
    return mass * 9.81 * height;
}

//上面代码就可以做如下重构
static final double GRAVITATIONAL_CONSTANT;
double potentialEnergy(double mass,double height){
    return mass * GRAVITATIONAL_CONSTANT * height;
}
```



10：`Encapsulate Field 封装字段`

你的类中存在一个public字段，将它声明为private,并提供专门的访问函数。

```java
public String _name;

private String_name;

public String getName(){
    return name;
}
public void setName(String name){
    this.name = name;
}
```

日常开发中，要注意变量不要随便申明为public，如果这样的话，就破坏了封装性，既在外部可以随意访问该变量。



11：`Encapsulate Collection 封装集合`

有个函数返回集合，让这个函数返回该集合的一个只读副本，并在这个类中提供添加/移除集合元素的函数。

这种情况怎么说呢，一般可以考虑直接返回一个不可变的集合。jdk和guava都有提供对应的api，但具体的实现还是有区别的，一个是浅拷贝，一个是深拷贝。



12：`Replace Record With Data Class 以数据类取代记录`



13：`Replace Type Code With Class 以类取代类型码`

假如现在你类中，有一个数值类型码，但它并不影响类的行为，此时可以以一个新的类替换该数值类型码。



14：`Replace Type Code With SubClass 以子类取代类型码`

你有一个不可变的类型码，他会影响类的行为，以子类取代这个类型码，





#### 简化条件表达式

在平时开发中，难免会遇到一些比较复杂的条件表达式，如何将一个复杂的条件逻辑表达式分成若干个小块呢。



`1：Decompose Conditional 分解条件表达式`

假如有一个复杂的表达语句，那么是否可以考虑从if then else段落中提炼出独立的函数来判断呢。

```java
if(data.before(SUMMER_START) || data.after(SUMMER_END)){
    charge = quantity * _winterRate + _winterServiceCharge;
}else{
    charge = quantity * _summerRate;
}

// 是否可以重构为下面的代码呢
if(notSummer(date)){
    charge = winterCharge(quantity);
}else{
    charge = summerCharge(quantity);
}
```



2：`Consolidate Conditional Expression 合并条件表达式`

假如有一些列条件测试，都得到相同的结果，将这些测试合并为一个条件表达式，并将这个条件表达式提炼成一个独立的函数。

```java
double disabilityAmount(){
    if(_senitory < 2){
        return 0;
    }
    if(_monthsDisabled > 12){
        return 0;
    }
    if(_isPartTime){
        return 0;
    }
}

//是否可以考虑重构为
double disabilityAmount(){
   if(isNotEligibleForDisablity())
       return 0;
}

```



3：`Consolidate Duplicate Conditional Fragments 合并重复条件的条件片段`

假如在条件表达式的每个分支上有着相同的一段代码。那么可以考虑将这段重复的代码移到条件表达式之外。

```java
if(isSpecilaDeal()){
    total = price * 0.95;
    send();
}else{
    total = price * 0.8;
    send(); 
}

//重构为
if(isSpecilaDeal()){
    total = price * 0.95;
}else{
    total = price * 0.8;
}
send();
```



4：Remove Control Flag 移除控制标记

在系列的布尔表达式中，某个变量带有“控制标记”的作用，可以考虑能否break或者return语句来替换。

```java
void checkSecurity(String[] people){

	boolean found = false;

	for (int i = 0; i < people.length; i++){

        if (!found){

            if (people[i].equals("Don")){

                sendAlert();

                found = true;

            }

        	if (people[i].equals("John")){

            	sendAlert();

       		 	found = true;
            }
		}
}
        
//是否可以重构为下面代码呢
void checkSecurity(String[] people){

    for (int i = 0; i < people.length; i++){

        if (people[i].equals("Don") || people[i].equals("John")){

        	sendAlert();

   			break;
        }
    }
}

```



5：`Replace Nested Conditional With Guard Clauses 以卫语句取代嵌套条件表达式`

函数中的条件逻辑使人难以看清正常的执行路径。使用卫语句表现所有特殊情况。

```java
double getPayAmount(){

    double result;

    if (_isDead)
        result = deadAmount();

    else {

   		if (_isSeparated)
            result = separatedAmount();

   		 else {

    		if (_isRetired)
                result = retiredAmount();

   			 else 
                 result = normalPayAmount();	
         }
    }
    return result;
}

//是否可以重构为
double getPayAmount(){

    if (_isDead) return deadAmount();

    if (_isSeparated) return separatedAmount();

    if (_isRetired) return retiredAmount();

    return normalPayAmount();

}
```



6：`Replace Conditional With Polymorphism 以多态取代条件表达式`

你手上有个条件表达式，它根据对象类型的不同而选择不同的行为。

将这个条件表达式的每个分支放进一个子类内的覆写函数中，然后将原始函数声明为抽象函数。

```java
double getSpeed(){
    switch(_type){
        case EUROPEAN:
            return getBaseSpeed();
        case AFRICAN:
            return getBaseSpeed()- getLoadFactor()* _numberOfCoconuts;
        case NORWEGIAN_BLUE:
            return (_isNailed)? 0 : getBaseSpeed(_voltage);
    }
    throw new RuntimeException("Should be unreachable");
}

//其实这种情况一般可以考虑用枚举来处理，或者策略模式
```

7：Introduce Null Object 引入Null对象

你需要再三检查某对象是否为null。将null值替换为null对象。看案例吧

```java


class Customer{
    public String getName(){...}

   	public BillingPlan getPlan(){...}

    public PaymentHistory getHistory(){...}
}

   
Customer customer = site.getCustomer();

BillingPlan plan;

if (customer == null)plan = BillingPlan.basic();

else plan = customer.getPlan();

...

String customerName;

if (customer == null)customerName = "occupant";

else customerName = customer.getName();

...

int weeksDelinquent;

if (customer == null)weeksDelinquent = 0;

else weeksDelinquent = customer.getHistory().getWeeksDelinquentInLastYear();
```

对于上面这种问题该怎么去重构呢，首先是否考虑引用一个空的Customer对象，并且对上面方法提供默认的实现，那么在上面是否就可以少去对null的判断

```java
class NullCustomer extends Customer implements Null{
    public boolean isNull{
        return true;
    }
    
    String getName(){
        return "occupant";
    }
	
   	Plan getPlan(){
      return  BillingPlan.basic();
	}
}

class Customer{
    static Customer newNull(){
		return new NullCustomer();
	}
}

class Site...

    Customer getCustomer(){

    	return (_customer == null) ? Customer.newNull():_customer;

}
```



8：`Introduce Assertion 引入断言`

某一段代码需要对程序状态做出某种假设。以断言明确表现这种假设。这个平时还是可以看到，但自己比较少用。

```java
double getExpenseLimit(){

	// should have either expense limit or a primary project

	return (_expenseLimit != NULL_EXPENSE) ? _expenseLimit: _primaryProject.getMemberExpenseLimit();

}

double getExpenseLimit(){

    //这里可以预先判断bug
	Assert.isTrue (_expenseLimit != NULL_EXPENSE || _primaryProject != null);

	return (_expenseLimit != NULL_EXPENSE) ? _expenseLimit: _primaryProject.getMemberExpenseLimit();

}
```

注意，不要滥用断言。请不要使用它来检查“你认为应该为真”的条件，请只使用它来检查“一定必须为真”的条件。滥用断言可能会造成难以维护的重复逻辑。





#### 简化函数调用

在面向对象编程中，接口是一个很重要的概念，然后接口的调用，也就是函数的调用。在开发中，必须明确的将“修改对象状态”的函数和“查询对象状态”的函数区分开，如果混在一起，就会很麻烦。

1：`Rename Method 函数改名`

这个其实在平时也遇到过，就是如果一个函数名没有完全的表述这函数的用途，那么就需要进行修改。



2：`Add Parameter 添加参数`

如果某个函数，需要从调用端得到更多的信息，那么可能就会需要对函数添加参数。



3：`Remove Parameter 移除参数`

如果函数本体不再需要某个参数，那么久应该考虑将其删掉。相必，大家在平时，看到多余的参数可能也不想去删除，觉得没什么影响。其实，对于函数的参数来说，就表明了这个函数需要的信息，其他人调用的时候，就必须要去考虑参数的用途，所以多余的必须要删除掉。



4： `Separate Query from Modifier 将查询函数和修改函数分开`

某个函数既返回对象状态值，又修改对象状态。建立两个不同的函数，其中一个负责查询，另一个负责修改。但确实有时候有查询再复制的操作，那怎么整呢。

下面是一条好规则：任何有返回值的函数，都不应该有看得到的副作用。应该是说查询的函数，不应该有修改操作。



5：`Parameterize Method 令函数携带参数`

若干函数做了类似的工作，但在函数本体中却包含了不同的值。建立单一函数，以参数表达那些不同的值。



6：`Replace Parameter With Explicit Methods 以明确函数取代参数`

你有一个函数，其中完全取决于参数值而采取不同行为。针对该参数的每一个可能值，建立一个独立函数。

```java
void setValue(String name,int value){
    if(name.equals("hegith")){
        _height = value;
    }
    if(name.equals("width")){
        _width = value;
    }
}

//重构为
void setHeight(int value){
    this._height = value;
}

void setWidth(int value){
    this._width = value;
}
```



7：`Preserve Whole Object 保持对象完整`

你从某个对象中取出若干值，将它们作为某一次函数调用时的参数。改为传递整个对象。



8：`Replace Parameter With Methods 以函数取代参数`

对象调用某个函数，并将所得结果作为参数，传递给另一个函数。而接受该参数的函数本身也能够调用前一个函数。

让参数接受者去除该项参数，并直接调用前一个函数。

```java
int basePrice = _quantity * _itemPrice;

discountLevel = getDiscountLevel();

double finalPrice = discountedPrice (basePrice,discountLevel);  //这种传递是不是不太合适 

//重构为
int basePrice = _quantity * _itemPrice;

discountLevel = getDiscountLevel();

double finalPrice = discountedPrice  (basePrice);// 在这里面去通过方法获取discountLevel就可以了，没必要像上面那样传递过去

//最后整合一下 是不是可以这样
public double getPrice(){

return discountedPrice ();

}

private double discountedPrice (){

    if (getDiscountLevel()== 2)return getBasePrice()* 0.1;

    else return getBasePrice()* 0.05;

    }

    private double getBasePrice(){

    return _quantity * _itemPrice;

}
```

9：`Introduce Parameter Object 引入参数对象`

假如某些参数总是很自然地同时出现。以一个对象取代这些参数。

```java
void between(int start,int end){
    //假如这组参数很多地方用到
}

//是否可以考虑把这两个参数封装成一个对象
void between(IntRange range){
    
}
```

10：`Remove Setting Method 移除设值函数`

类中的某个字段应该在对象创建时被设值，然后就不再改变。去掉该字段的所有设值函数。

这种情况就不多说了，有些字段是不能修改，就不能暴露修改方法。



11：`Hide Method 隐藏函数`

有一个函数，从来没有被其他任何类用到。将这个函数修改为private。

也就是没有被本身类以外用到，就可以设置为private。



12：`Replace Constructor with Factory Method 以工厂函数取代构造函数`

你希望在创建对象时不仅仅是做简单的建构动作。将构造函数替换为工厂函数。

这个就不多说了，平时见的也比较多。



13：`Encapsulate Downcast 封装向下转型`

某个函数返回的对象，需要由函数调用者执行向下转型（downcast）。将向下转型动作移到函数中。

```java
Object lastReading(){
    return readings.lastElement();
}

//是否可以改一下呢
Reading lastReading(){

	return (Reading)readings.lastElement();

}
```



14：`Replace Error Code with Exception 以异常取代错误码`

某个函数返回一个特定的代码，用以表示某种错误情况。改用异常。现在项目中，也是通过异常取代错误码，但可以把错误码设置在异常中，然后统一铺货该异常，处理里面错误码就OK。

```java
int withdraw(int amount){
    if(amount > _balance){
        return -1;
    }else{
        _balance -= amount;
        return 0;
    }
}

//重构为
void withdraw(int amount)throws BalanceException {

    if (amount > _balance)
        throw new BalanceException();

    _balance -= amount;

}
```

但这个里要注意的是，具体抛出的是受控异常还是非受控异常。



15：`Replace Exception with Test 以测试取代异常`

面对一个调用者可以预先检查的条件，你抛出了一个异常。修改调用者，使它在调用函数之前先做检查。

```java
double getValueForPeriod(int periodNumber){

    try {
        return _values[periodNumber];

    } catch (ArrayIndexOutOfBoundsException e){

        return 0;

    }
}

//上面看着是不是怪怪的 为何不直接在里面做检查呢
double getValueForPeriod(int periodNumber){

    if (periodNumber >= _values.length)return 0;

    return _values[periodNumber];

}
```




---
layout:     post
title:      序列化工具
subtitle:   fastJson和Gson
date:       2020-03-17
author:     tryingpfq
header-img: img/post-bg-serialize1.jpg
catalog: true
tags:
    - serialize
---

### Serialize

> json序列化和反序列化，这个是日常中比用到的东西，而且现在工具也比较多，但常用的还是fastJson，毕竟速度快。这主要是关于fastJson和Gson使用时候遇到的一些问题



### FastJSON

* 序列化

  要注意的是，如果被序列化的对象中，有字段没有对应的Get方法，那么该字段是不会被序列化的。具体原因可以看下源码。

  ~~~java
  //JSONSerialize
  public final void write(Object object) {
          if (object == null) {
              out.writeNull();
              return;
          }
          Class<?> clazz = object.getClass();
      	//断点断不进去，不知道为啥，感觉是会生成一个 AsmSerializer_1_*的代理 
          ObjectSerializer writer = getObjectWriter(clazz);
          try {
              writer.write(this, object, null, null, 0);
          } catch (IOException e) {
              throw new JSONException(e.getMessage(), e);
          }
      }
  ~~~

  那如果我们有些字段是不需要序列化的时候，该怎么做呢，其实在字段面前加上 transient关键字就好了

* 反序列化

  我经常会遇到这种问题，就是反序列化的时候，对象和序列化时候对象字段不一致问题。

  反序列化的时候，要有无参构造器，否则会反序列化失败报错。

  1：反序列化的时候，如果字段缺少对应的set方法，该字段是无法被反序列化的。

  2：如果反序列化和序列化的时候，字段不一致，是可以成功反序列化的，只是会缺少字段和缺少字段的赋值，但会缺少该字段。

* 如何定制复杂的序列化和反序列化

  这里就直接上代码吧，我们要用到Java 反射的Type

  ~~~java
   private static final TypeReference<Map<Integer,Info>> integerToInfo = new TypeReference<Map<Integer,Info>>(){};
  
    private static String data1 = "{1:{\"id\":1,\"map\":{1111:\"abc\"},\"name\":\"aaa\"}}";
  
   Map<Integer, Info> map = JSON.parseObject(data1, integerToInfo.getType());
  
  ~~~

  这就是简单的应用呀，具体可以看Json里面接口源码，还是很多的，也比较复杂，但还是可以稍微看一下哦

  ~~~java
  com.alibaba.fastjson.serializer.SerializeConfig#createJavaBeanSerializer(java.lang.Class<?>)
  ~~~

  这方法里面有个buildBeanInfo方法，`com.alibaba.fastjson.serializer.SerializeConfig#fieldBased`默认值是false，最后给字段进行赋值调用的是

  `com.alibaba.fastjson.util.TypeUtils#computeGetters`方法，这里面具体做的事情，就是根据反射获取所有以get开头的方法(当然里面部分方法也要过滤)再去除get，小写化后的值作为key，get()的值作为value,拼接到json串中。所以这里也解释了fastjson默认是根据get 和 set 方法进行序列化和反序列化的。

  如果我们将属性`com.alibaba.fastjson.serializer.SerializeConfig#fieldBased`设置为true的话，就不是根据get和set方法来进行序列化和反序列化。字段集合会调用这个方法：`com.alibaba.fastjson.util.TypeUtils#computeGettersWithFieldBase`

  

  **但不知道为啥，断点一直断不进去上面写的方法?**

  

  ### GSON

   Google的Gson是目前功能最全的Json解析神器，但好像性能上要比fastJson差一些。

  1：对于匿名内部类来说，gson可能出现问题哦
  
  ​	比如在初始化map的时候，有时候会通过这种方式来优雅代码，特别是对于Java8中使用lamda表达式（ConcurrentHashMap 中死循环问题）
  
  ```
  private static Gson gson = new Gson();
  
  Map<String, String> map = new HashMap<String, String>() {
      {
  		put("abc", "abc");
    }
  };
System.out.println(gson.toJson(map));
  ```

  这里会遇到一个奇怪的问题，就是打印的值为 null，看了下里面源码，com.google.gson.internal.Excluder.

  create()，方法中，会判断是不是匿名内部类，好像是这样的，但是这个方法excludeClassChecks(rawType) -> isAnonymousOrLocal(clazz)会返回true。上面初始化map的时候，会创建一个匿名内部类，大概就是这个原因吧。
  
  2：直接看代码问题
  
  ~~~java
  private static Gson gson = new Gson();
  
  Map<String, String> map = new HashMap<>();
map.put("abc","abc");
  Set<Map.Entry<String, String>> entries = map.entrySet();
System.out.println(gson.toJson(entries));
  ~~~
  
  这里输出的结果是[{}]，那就是没有成功反序列化为json串，字段值没有被打印出来，为什么呢，那只能看源码，很有可能就是字段值没有进行赋值哦，那具体是什么。
  
  ~~~java
  com.google.gson.internal.bind.ReflectiveTypeAdapterFactory#getBoundFields
  其实这里就是绑定字段呀
  
  Map<String, BoundField> result = new LinkedHashMap<String, BoundField>();
if (raw.isInterface()) {//raw是“集合元素”的类型，为java.util.Map.Entry，是一个接口！
      return result;
}
  ~~~

  所以用的时候要注意一个问题就是，集合中的泛型不要使用接口类型，不然会序列化失败。

  问题用fastjson是不有问题的。

  ### 比较

   加入出现属性的set方法和字段名称不一致，两者会出现什么情况呢
  
  **序列化的时候**
  
  ~~~java
  		//info中的字段名声 firstName，方法是setName();
  		Info info = Info.valueOf(); 
          String str = JSON.toJSONString(info);
          System.err.println("fastJson result: " + str);

          str = gson.toJson(info);
        System.err.println("gson result: " + str);
  ~~~
  
看下输出来的结果：
  
fastJson result: {"id":1,"map":{1111:"abc"},"name":"aaa"}
  gson result: {"id":1,"firstName":"aaa","map":{"1111":"abc"}}

  

  **为什么会这样呢？**

  *序列化和反序列化时fastJson是按照get和set方法的名字来转换的，而gson则是按照属性的名字来转换的。*

  

  所以可能会出现这种问题，对于fastJson来说，如果字段名不变，序列化前后某个字段的get方法和set方法名称修改了的话，就会造成问题。而gson是不会的。**你有没有遇到过这种问题呢**

  
  
  那对应的能成功的反序列化吗？写个测试就可以了
  
  ~~~java
   String fastJson = "{\"id\":1,\"map\":{1111:\"abc\"},\"name\":\"aaa\"}";
   String gsonStr = "{\"id\":1,\"firstName\":\"aaa\",\"map\":{\"1111\":\"abc\"}}";
  
   Info info = JSON.parseObject(fastJson, Info.class);
 System.err.println(info.toString());
   info = gson.fromJson(gsonStr, Info.class);
 System.err.println(info.toString());
  ~~~
  
输出的结果：
  
Info{id=1, firstName='aaa', map={1111=abc}, other='null'}
  Info{id=1, firstName='aaa', map={1111=abc}, other='null'}

  可见还是对应的都能够把数据成功反序列化。
  
  
  
  
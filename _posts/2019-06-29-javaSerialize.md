---
layout:     post
title:      Java序列化
subtitle:   Java自带的序列化。
date:       2019-06-29
author:     tryingpfq
header-img: img/post-bg-serialize.jpg
catalog: true
tags:
    - 序列化
---

> 序列化在现在工程开发中，是无处不在的，特别是对于现在分布式系统，如果必须有一套好的序列化机制，不然会影响网络的传输，然后大大影响效率。比如有基于XML（WebService框架）、基于JSON，Dubbo是基于hessian2、google的Protocol Buffers...

### 前言 
Java本身有自己的序列化机制，只要某个类实现了Serialize接口，则表明这个类是可以被序列化的。而且对于每一个实现该接口的类，都会有一个uuid,用来唯一标识某一个对象，以便于进行序列化和反序列化，下面主要是有关java本身序列化过程。


### Java序列化存在的问题
* 1：序列化后的数据结果比较大，传输效率比较低下。
* 2：不能跨语言进行对接。(你可以想一下，如果是基于Java本身的一个序列化的数据传输过去，到另一个应用服务上的开发语言并不是Java，那就无法对那部分字节进行反序列化)。

### Java序列化过程
比如我们要对一个Person类进行序列化，属性，这个类要实现Serialize接口。下面可以看一段代码。



public class JavaSerial {

    public static void main(String[] args) throws IOException {
        seriallize();
        //deSerialize();
    }

    public static Person deSerialize(){
        try {
            ObjectInputStream oi = new ObjectInputStream(new FileInputStream("person"));

            Person person = (Person) oi.readObject();

            return person;
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        return null;
    }


    public static void seriallize() throws IOException {
        ObjectOutputStream oo = null;
        try {
            oo = new ObjectOutputStream(new FileOutputStream("person"));
            Person person = new Person();
            person.setName("peng");
            person.setAge(26);

            oo.writeObject(person);

            ObjectInputStream oi = new ObjectInputStream(new FileInputStream("person"));


            Person person1 = (Person) oi.readObject();
            System.out.println(person1);
            person.setName("peng1");

            oo.flush();
            oo.writeObject(person);

            Person person2  = (Person) oi.readObject();
            System.out.println(person2);

            System.out.println(person1 == person2);

            System.out.println(person1.equals(person2));

        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } finally {
            if (oo != null) {
                oo.close();
            }
        }

    }


     static class Person implements Serializable{

        private static final long serialVersionUID = -3218838394429987427L;

        private String name;

        private int age;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

         @Override
         public String toString() {
             return "Person{" +
                     "name='" + name + '\'' +
                     ", age=" + age +
                     '}';
         }
     }

}


上面就是对一个类进行序列化和反序列化的一个简单过程。

### Serialize序列化注意点
* 1：如果我们有一个父类，若其子类要进行序列化，并且想同时序列化父类的属性，则父类和子类都必须实现Serialize接口。

* 2：序列化的存储规则，对同一个对象进行多次写入，打印出的第一次存储结果和第二次存储结果，只多了5个字节的引用关系，并不会导致文件累加，这也就是Java序列化本身机制的一种优化策略。

* 3：静态变量的序列化，如果某个类中，存在静态变量，序列化前的值为123，当序列化完后，将这个值修改为321，那么反序列化出来的值是修改前的值，也就是说序列化并不保存静态变量的状态。

* 4：transient关键字，这个关键字应该大家都用过，就是忽略某个字段不进行序列化，那么反序列化的时候，这个字段的值就是一个默认值。

### serialVersionUID
文件流中的class和classpath中的class，也就是修改过后的class，不兼容了，处于安全机制考虑，程序抛出了错误，并且拒绝载入。从错误结果来看，如果没有为指定的class配置serialVersionUID，那么java编译器会自动给这个class进行一个摘要算法，类似于指纹算法，只要这个文件有任何改动，得到的UID就会截然不同的，可以保证在这么多类中，这个编号是唯一的。所以，由于没有显指定 serialVersionUID，编译器又为我们生成了一个UID，当然和前面保存在文件中的那个不会一样了，于是就出现了2个序列化版本号不一致的错误。因此，只要我们自己指定了serialVersionUID，就可以在序列化后，去添加一个字段，或者方法，而不会影响到后期的还原，还原后的对象照样可以使用，而且还多了方法或者属性可以用。



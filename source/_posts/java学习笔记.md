---
title: "java学习笔记"
onlyTitle: true
date: 2022-05-10 13:05:36
categories:
- java
- java杂谈
tags:
- java杂论
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B816.jpg
---



# java学习简记

学习CC链过程中发现java水平不够，本笔记主要介绍java我个人认为比较重要或者困难的地方。

## 类定义

abstract声明抽象类和抽象方法（没有任何实现的类和方法），为了以后扩充。如果类有抽象方法，那该类也必须为抽象类

extends继承父类,implements定义接口

synchronized定义方法只能同时被一个线程访问

transient修饰变量，在序列化时忽略该变量

volatile修饰的成员变量在每次被线程访问时都要重读值，也就是两个线程访问到的都是同一值。同时也组织了编译器对该变量进行优化

Character传char参数会自动转换为Character对象，叫做装箱。装箱后数据类型为对象，可以使用对应的许多方法

String定义的字符串对象，定义后不可改变。改变的话需要用StringBuffer或者StringBuilder。String.format()添加格式化字符串（带参数的字符串）

处理数组时可以使用循环或者foreach循环。util.Arrays类也提供了赋值、排序、比较、查找等方法

* `for(double element:List){}`循环遍历List数组里的double类型元素，该元素在每次循环用element命名使用

## 正则匹配：

* 比如w3Cschool里的代码：

```java
String line = "This order was placed for QT3000! OK?";
      String pattern = "(.*)(\\d+)(.*)";

      // 创建 Pattern 对象
      Pattern r = Pattern.compile(pattern);

      // 现在创建 matcher 对象
      Matcher m = r.matcher(line);
      if (m.find( )) {
         System.out.println("Found value: " + m.group(0) );
         System.out.println("Found value: " + m.group(1) );
         System.out.println("Found value: " + m.group(2) );
      } else {
         System.out.println("NO MATCH");
      }
```

按照捕获组的思路，group(0)为`(.*)(\\d+)(.*)`，`(.*)`表示除了换行外的任意字符串。`\\`转义为`\`，`\d`匹配数字，`+`匹配以+前面开头的字符串，比如"zo+"匹配"zo"和"zoo"。

所以`(\\d)`匹配到的值为3000，`(\\d+)`匹配到的值为0，`(.*)(\\d+)(.*)`匹配到的值为QT3000! OK? 即group(0)的值

group(1)为`(.*)`，值为QT300

group(2)为`(\\d+)`，值为0

* start()返回匹配字符串的起始字符在整个被匹配字符串的位置，end()则是结束位置。count返回匹配成功的次数
* 除此以外还有matches整串匹配，lookingAt部分匹配。replaceFirst首次替换，replaceAll全部替换。appendReplacement正则替换，appendTail匹配完成的字符串返回给对象

# 面向对象

* 可变参数：`...`作最后一个参数时表示可接受多个参数
* 回收对象：`finalize()`指定对象销毁时执行的操作，要用protected限定以免被类外调用。销毁对象可以在类方法里调用`System.gc()`，该方法将回收它所定义的“垃圾”。finalize一般以`super.finalize()`的方式使用（超类终结器）

### 流

* 控制台输入流：

`BufferedReader bi = new BufferedReader(new InputStreamReader(System.in));`

将System.in包装在BufferReader对象中创建字符流。对字符流存在很多操作。比如read()方法读取输入的字符，或者用readLine()读取字符串（enter为数组下一索引），读取结束会返回IOException

* 控制台输出流

`System.out.write(字符串or流)`输出流。System.out是PrintStream类对象的引用，PrintStream继承OutoutStream并实现了write()。所以该方法的作用和println()和print()一样。

* 文件读取流

`InputStream f = new FileInputStream("文件路径")`也可以向字符串的位置填一个文件对象。创建输入流对象读取文件

* 文件写入流

`OutputStream f = new FileOutputStream("文件路径")`写入内容，同理也能填文件对象

* Scanner数据输入

控制台输入流操作起来不方便，可以用`Scanner scan = new Scanner(System.in)`接收数据，再使用Scanner类的方法。而next和next的方法细节请看`https://www.w3cschool.cn/java/java-scanner-class.html`

## 异常处理

学过python的话，我记得try...except...涉及异常处理，C++和PHP里则是try...catch...。都是try里的函数可能涉及异常，出现异常则throw一种Exception，catch到该类Exception进行处理。java也有使用try...catch进行异常处理的，只不过没有定义方法时进行异常处理常用。

java内置了异常类在java.lang包，所以可以直接使用。比如数组越界异常ArrayIndexOutOfBoundsException。异常类通常接在方法声明中，比如`public void a(double b) throws RemoteException{}`，throw异常类会自动捕捉并处理异常

* finally放在`try...catch...`的最后，是运行完`try...catch...`必定会处理的内容（不一定非要finally结尾)
* 自定义异常。`class AException extends Exception{}`，定义AException异常，同时可以重构其构造函数来对异常进行处理。

## 继承

所有类都是从java.lang.Object类继承来的，除了Object外所有类都必须有一个父类

* extends继承的子类有父类的所有成员变量，但是不能访问父类的private成员。子类可以重写父类的任何方法，但是通常必须满足：1.参数列表相同 2.返回类型是父类派生类 3.访问权限大于等于父类该方法 4.final、static不能被重写。等要求（菜鸟使用就完全一样的声明方式，别记）

* instanceof检查A是否为B的子类

* java不能同时继承多个类，但是可以同时实现多个接口

子类调用父类**被重写**方法时用super，比如下列代码Dog就是调用了父类Animal的move。如果是构造函数，直接`super();`就能调用

```java
class Animal{

   public void move(double a){
      System.out.println("动物可以移动");
   }
}

class Dog extends Animal{

   public void move(double a){
      super.move(double a); // 应用super类的方法
      System.out.println("狗可以跑和走");
   }
}
```

### 多态

java所有对象都有多态性，比如`public class Deer extends Animal implements Vegetarian{}`就同时具有Animal、vegetarian、Deer、Object的性质。

这里都不用管，虚方法那些也不用学

## 抽象类&方法

抽象类就是定义了类，但是没有实现。抽象类用abstract定义

因为没有实现，抽象类的实例化自然也没有意义。抽象类存在的意义就是让子类继承，并实现其中的部分or全部方法

> 如果一个类包含抽象方法，该类必须是抽象类。而且子类继承时必须将抽象方法全部实现。否则子类为抽象类

## 接口

接口不能生成对象。继承接口的类必须实现接口内所有方法（不然就是抽象类），除此以外：

1. 接口没有构造方法

2. 接口内所有方法都必须是抽象方法（接口隐式抽象，也就是声明方法不需要用abstract指定抽象方法，同理声明接口也一样)。接口方法都是公有

3. 接口不能包含除static、final之外的成员变量

用`public interface 接口名( extends 类)`的方式声明接口。

用`public class 类名 implements 接口名[,...多个接口]`的方式实现接口。

> 实现接口时不能抛出强制性异常，只能在父类或者父接口中抛出该强制性异常

接口的继承也使用extends，同理在实现接口时需要实现该接口及其父类接口的全部方法。

## package包

比如下列代码的路径就是org.example.A.java

```java
package org.example
    public class A{}
```

同样，有一个代码是：

```java
package org.example
    public class B{}
```

那么在使用的时候import org.example.*;或者import org.example.A;import org.example.B;就能同时使用A、B两个类

正因通过路径使用包的方式，A、B就必须存放在工作目录的org/example目录下



## java数据结构

### 枚举

Enumeration 定义枚举数据。其实枚举就像一个一次性数组。比如w3school给出的代码：

```java

   public static void main(String args[]) {
      Enumeration days;
      Vector dayNames = new Vector();
      dayNames.add("Monday");
      dayNames.add("Tuesday");
      dayNames.add("Wednesday");
      dayNames.add("Thursday");
      dayNames.add("Friday");
      days = dayNames.elements();
      while (days.hasMoreElements()){
         System.out.println(days.nextElement()); 
      }
   }
}
```

利用Vector容器动态存放字符串数组。用element返回vector中元素枚举（队列形式，不知道队列和栈的转战数据结构）

hasMoreElements判断容器中还有无枚举。nextElement()返回下一个枚举元素

### 位集合(Bitset)

关于C++bitset可以去看我的sha256算法的bitset实现https://blog.csdn.net/qq_51796436/article/details/124999060?spm=1001.2014.3001.5502。其实bitset就是一个bool数组，只不过对bool值的实现做了很多优化

### 向量(vector容器)

动态数组。和ArrayList的主要区别是vector包含了很多不属于集合框架的方法。用new的方式新建容器，用调用方法的方式使用容器，比如有以下常用的方法：

* elements()返回容器枚举
* hashCode()返回向量哈希码
* toString()返回向量的字符串形式（每个元素的String）

### 栈

后进先出。是vector的子类，所以能使用vector的方法。定义方式和vector容器一样:

`Stack a = new Stack();`

push(a)压栈，pop()出栈，peek()查看栈顶部但不进行出栈操作

### 哈希表&HashMap（关键）

Hashtable和HashMap一样，只是Hashtable支持同步。可以理解为javascript语言的命名索引。用key-value的方式组成数组，一个key对应一个value。

### 属性（Properties接口）

Properties继承于Hashtable，只不过key-value都是字符串

## 泛型

所谓泛型也就是同时指代了多个数据类型。

```java
public static < E > void printArray( E[] inputArray )
   {
      // 输出数组元素            
         for ( E element : inputArray ){        
            System.out.printf( "%s ", element );
         }
         System.out.println();
    }
```

上述代码定义了一个函数输出数组元素，泛型参数的声明为`<E>`,E[]就指代了任意类型的数组。因为通常泛型的返回值也是泛型，所以声明方法时也就进行泛型声明。

再看一个泛型方法的定义：

```java
public static <T extends Comparable<T>> T maximum(T x, T y, T z)
   
```

用extends继承的方式定义上界，那什么是上界呢？

在数学上来看，泛型就意味着定义域R。但一般来说接收的参数可能只要数字number或者正数域这些相对狭窄的域，就需要一个上界缩小需要值的范围

上述定义规定了泛型T 上界为`Comparable<T>`。查看Compareable的源码可以看到该接口接收参数也为泛型。`<T extends Comparable<T>>`定义完泛型T后要紧接一个T，没定义不用接

有泛型方法就有泛型类，泛型类的定义和使用如下：

```java
public class Box<T> {}//定义
Box<String> stringBox = new Box<String>();//main中的使用
```

## 序列化

序列化的目的就是方便存取。

序列化一般是writeObject，反序列化一般是readObject。

* 可以序列化的类必须满足：**必须实现了java.io.Serializable对象**

序列化只需要以下代码：

```java
FileOutputStream fileOut = new FileOutputStream("类路径");
ObjectOutputStream out = new ObjectOutputStream(fileOut);
out.writeObject(e);
out.close();
fileOut.close();
```

将文件流写入FileOutputStream流，然后将FileOutputStream写入到对象流ObjectOutputStream中，然后进行序列化

同理，反序列化：

```java
FileInputStream fileIn = new FileInputStream("类路径");
ObjectInputStream in = new ObjectInputStream(fileIn);
e = (Employee) in.readObject();
in.close();
fileIn.close();
```

需要用(Employee)指定入口类。这样序列化e就成了类引用

## java网络编程

一想起网络编程这四个字就应该想起套接字，也就是ip:port的形式。直接来看一个最简化的客户端：

```java
// 文件名 GreetingClient.java

import java.net.*;
import java.io.*;

public class GreetingClient
{
   public static void main(String [] args)
   {
      String serverName = args[0];
      int port = Integer.parseInt(args[1]);
      try
      {
         Socket client = new Socket(serverip, port);
         OutputStream outToServer = client.getOutputStream();
         DataOutputStream out = new DataOutputStream(outToServer);
		 out.writeUTF("Hello from "+ client.getLocalSocketAddress());
         InputStream inFromServer = client.getInputStream();
         DataInputStream in = new DataInputStream(inFromServer);
         System.out.println("Server says " + in.readUTF());
         client.close();
      }catch(IOException e)
      {
         e.printStackTrace();
      }
   }
}
```

定义一个socket对象。

getOutputStream()即接收来自服务端的数据，并转化为OutputStream流写入到DataOutputStream，再调用方法输出

getInputStream输入消息，创建数据对象DataInputStream，并把要发送的消息写入到数据流。

服务端同理。只不过用serverSocket.accept()接收套接字连接请求



## 多线程

java实现多线程有三种方式

1. 继承Runnable接口（`implements Runnable`)
2. 继承Thread类（`extends Thread`)
3. Callable和Future结合使用，并实现call()方法

通过`new 线程()`的方式在main函数创建新线程。定义线程的同时需要重写构造函数。

* 线程本身为主线程，start会生成子线程;一个start开启一个子线程，子线程会默认执行一遍run()

```java
class NewThread extends Thread {
   NewThread() {
      // 创建第二个新线程
      start(); // 开始线程
   }
public void run(){}
```

前面两个都要重写run()方法作为线程执行的主体（Runnable非必要）。











## 其他内容

java的编译器一般下Intellij IDEA（专业版，非专业版没有很多后续学习需要的东西）。

java可能会需要很多版本，比如jdk8u65\jdk8u111\jdk8u200以上等，可以同时存放在一个目录，需要用到的时候在IDEA项目结构里切换SDK

java8版本也叫1.8

装完java发现你的burpsuite 、菜刀等工具崩了，改回以前的环境吧。环境变量改一下就行

moven只是一种管理工具

IDEA双击shift查找类or方法

修改依赖通常只需要在pom.xml修改dependency然后右键->moven->重新加载就行





想学习java web安全的可以在先知社区看看我的java反射&Common-collections1&6。链接如下：后续应该会写很多java web

https://xz.aliyun.com/t/11861#toc-7


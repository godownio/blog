---
title: "jackson 反序列化(纯流程无POC)"
onlyTitle: true
date: 2024-10-11 15:03:23
categories:
- java
- 框架漏洞
tags:
- jackson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8105.png
---

pom.xml：

```xml
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.3</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.9.3</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>2.9.3</version>
        </dependency>
```



基础知识点：

https://drun1baby.top/2023/12/07/Jackson-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%EF%BC%88%E4%B8%80%EF%BC%89%E6%BC%8F%E6%B4%9E%E5%8E%9F%E7%90%86/

以下知识均来自上述链接，有空请移步仔细阅读

给个总结：

## 基础知识总结

jackson是用来序列化和反序列化json的解析器之一

Jackson 的核心模块由三部分组成。

- jackson-core，核心包，提供基于"流模式"解析的相关 API，它包括 JsonPaser 和 JsonGenerator。  Jackson 内部实现正是通过高性能的流模式 API 的 JsonGenerator 和 JsonParser 来生成和解析 json。
- jackson-annotations，注解包，提供标准注解功能；
- jackson-databind ，数据绑定包， 提供基于"对象绑定" 解析的相关 API （ ObjectMapper ）  和"树模型" 解析的相关 API （JsonNode）

## ObjectMapper

使用该对象的方法，可以解析JSON到java对象，也可以把java对象转化为JSON

### json转化为对象

将json转化为对象code：（需要有Person类）

```java
String json = "{\"name\":\"John\", \"age\":30}";
ObjectMapper objectMapper = new ObjectMapper();
Person person = objectMapper.readValue(json, Person.class);
```



### 对象转化为json

```java
		ObjectMapper objectMapper = new ObjectMapper();
        Person person = new Person();
        person.setAge(123);
        person.setName("fakes0u1");
		String jsonstring = objectMapper.writeValueAsString(person);
```

共有三种方法可以将对象转化为json：

- writeValue()
- writeValueAsString()
- writeValueAsBytes()



## JsonParser

ObjectMapper.readValue其实底层就是用的JsonParser进行解析。算是另一种更底层的转化方式，由于底层，所以速度更快

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011101628760.png)

### json转化为对象

JsonParser的使用如下：

```java
		String json = "{\"name\":\"fakes0u1\",\"age\":123}";
        JsonFactory jsonFactory = new JsonFactory();
		JsonParser parser = jsonFactory.createParser(json);
```

### 对象转化为Json

使用JsonGenerator从java对象生成JSON

```java
		JsonFactory jsonFactory = new JsonFactory();
        Person1 person1 = new Person1();
		JsonGenerator jsonGenerator = jsonFactory.createGenerator(new File("output.json"), JsonEncoding.UTF8);
        jsonGenerator.writeStartObject();
        jsonGenerator.writeStringField("name","fakes0u1");
        jsonGenerator.writeNumberField("age",123);
        jsonGenerator.writeEndObject();

        jsonGenerator.close();
```

也可以把文件流换成其他流进行写入



## JacksonPolymorphicDeserialization 机制

简单地说，Java 多态就是同一个接口使用不同的实例而执行不同的操作。

那么问题来了，如果对多态类的某一个子类实例在序列化后再进行反序列化时，如何能够保证反序列化出来的实例即是我们想要的那个特定子类的实例而非多态类的其他子类实例呢？—— Jackson 实现了 JacksonPolymorphicDeserialization 机制来解决这个问题。

JacksonPolymorphicDeserialization 即 Jackson  多态类型的反序列化：在反序列化某个类对象的过程中，如果类的成员变量不是具体类型（non-concrete），比如  Object、接口或抽象类，则可以在 JSON 字符串中指定其具体类型，Jackson 将生成具体类型的实例。

简单地说，就是将具体的子类信息绑定在序列化的内容中以便于后续反序列化的时候直接得到目标子类对象，其实现有两种，即 `DefaultTyping` 和 `@JsonTypeInfo` 注解。这里和前面学过的 fastjson 是很相似的。

### DefaultTyping

Jackson 提供一个 enableDefaultTyping 设置，该设置位于`jackson-databind-2.7.9.jar!/com/fasterxml/jackson/databind/ObjectMapper.java`

共有以下四个选项：

```
JAVA_LANG_OBJECT
OBJECT_AND_NON_CONCRETE
NON_CONCRETE_AND_ARRAYS
NON_FINAL
```

先用enableDefaultTyping进行设置，再调用writeValueAsString进行序列化。就会按照属性类型进行序列化和反序列化。看下case吧：

```java
ObjectMapper mapper = new ObjectMapper(); 
mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.JAVA_LANG_OBJECT);
String json = mapper.writeValueAsString(p);
```

左边是没设置enableDefaultTyping，右边是设置的

![compare](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/compare.png)

这四个选项能反序列化的属性范围如下：

| DefaultTyping类型       | 能反序列化的属性                                    |
| ----------------------- | --------------------------------------------------- |
| JAVA_LANG_OBJECT        | 属性的类型为Object                                  |
| OBJECT_AND_NON_CONCRETE | 属性的类型为Object、Interface、AbstractClass        |
| NON_CONCRETE_AND_ARRAYS | 属性的类型为Object、Interface、AbstractClass、Array |
| NON_FINAL               | 所有除了声明为final之外的属性                       |

**enableDefaultTyping()**默认无参数设置为OBJECT_AND_NON_CONCRETE

有的不理解interface和Object在这里的区别是什么：

按CC链来举例，Transformer[]是数组，Transformer是接口，而InvokerTransformer这种就是Object



## @JsonTypeInfo注解

和DefaultTyping功能类似，JsonTypeInfo是直接注解字段，达到相同的功能

比如在需要序列化和反序列化的对象内，标记Object字段：

```java
public class Person {  
    public int age;  
    public String name;  
    @JsonTypeInfo(use = JsonTypeInfo.Id.NONE)  
    public Object object;  
  
	getter...setter...
}
```

以下两种注解能达到反序列化

- JsonTypeInfo.Id.CLASS
- JsonTypeInfo. Id. MINIMAL_CLASS

比如注解为`@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS)`

序列化出来的json字符如下：

```json
{"age":6,"name":"drunkbaby","object":{"@class":"com.drunkbaby.Hacker","skill":"hiphop"}}
```

反序列化当然也能按照指定的类进行反序列化

注解为`@JsonTypeInfo(use = JsonTypeInfo. Id. MINIMAL_CLASS)`时，用 @c 替代了 @class，就是一个简单的缩短，和`JsonTypeInfo.Id.CLASS`作用相同

```
{"age":6,"name":"drunkbaby","object":{"@c":"com.drunkbaby.Hacker","skill":"hiphop"}}
```

## 总结

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



OK，下面看看反序列化的过程，到底怎么反序列化的

## 调试

### 调试代码

这里就只调试开启了enableDefaultTyping()的case，注解的方式过程也差不多

Sex接口：

```java
public interface Sex {
    public void setSex(int sex);
    public int getSex();
}
```

MySex类：

```java
public class MySex implements Sex {
    int sex;
    public MySex() {
        System.out.println("MySex构造函数");
    }


    public int getSex() {
        System.out.println("MySex.getSex");
        return sex;
    }

    public void setSex(int sex) {
        System.out.println("MySex.setSex");
        this.sex = sex;
    }
}
```

Person类：

```java
public class Person {
    public Person(){
        System.out.println("Person Constructor");
    }
    private String param1;
    private Sex sex;
    
    public String getParam1() {
        System.out.println("getParam1");
        return param1;
    }
    public void setParam1(String param1) {
        System.out.println("setParam1");
        this.param1 = param1;
    }
    public Sex getSex() {
        System.out.println("getSex");
        return sex;
    }
    public void setSex(Sex sex) {
        System.out.println("setSex");
        this.sex = sex;
    }
}
```

jacksoncase主类：

```java
public class jacksoncase {
    public static void main(String[] args) throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        mapper.enableDefaultTyping();

        String json = "{\"param1\":\"aaa\",\"sex\":[\"org.exploit.othercase.jackson.MySex\",{\"sex\":1}]}";
        Person p2 = mapper.readValue(json, Person.class);
        System.out.println(p2);
    }
}
```

输出如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011150540636.png)

可以看到是按照 `构造函数->setter赋值`，对里面的字段也采取这个嵌套的方式进行反序列化



### 调用构造器和setter

在readValue处打上断点

跟进到`_readMapAndClose`，从JsonParser中读取并解析JSON数据为指定的valueType对象

会走进deser.deserialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011154527112.png)

其中令牌可以看到JsonToken，`{}`代表对象，`[]`代表数组

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250418163443014.png)

我们这里JSONToken令牌是START_OBJECT，表示是个对象进行反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011154556761.png)

继续跟进，如果令牌是START_OBJECT，会走进if调用vanillaDeserialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011154900652.png)

p.nextToken就是`{`下一个符号，也就是我们的"param1"，对应的JsonToken是FIELD_NAME

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011155135088.png)

vanillaDeserialize就是jackson反序列化json至对象的主逻辑，如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011155829805.png)

_valueInstantiator包含了从valueType中取到的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011155926684.png)

createUsingDefault完成了实例化Person类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011160138840.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011160150855.png)

把这个实例化的空的bean，存到JsonParser中。

如果JsonParser当前处于字段名（FIELD_NAME）状态，则循环处理每个属性

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011160448411.png)

处理方式为：

* 获取当前属性名
* 获取类中对应的字段作为prop，包含了对应的setter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011161009399.png)

* prop不为空，则调用deserializeAndSet进行设置字段

跟进到deserializeAndSet内，继续跟到StringDeserializer.deserialize

我们设置的第一个键值对为`"param1":"aaa"`，所以字段值属于VALUE_STRING，直接返回了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011161301958.png)

返回后直接调用setter赋值了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011161415319.png)

再看下循环到的自定义的MySex对象，在vanillaDeserialize do...while的第二次循环：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011161536746.png)

由于`_valueTypeDeserializer`不为空，（在初始化的时候就设为了Array对应的反序列化器），于是调用指定反序列化器的deserializeWithType进行反序列化字段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011170025600.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011170008762.png)

跟进到`AsArrayTypeDeserializer._deserialize`，看名字也知道，数组的反序列化器，继续跟进发现又走到了vannillaDeserialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241011163746878.png)

也就是个嵌套的解析

继续跟到StdValueInstantiator.createFromString，如果当前类有构造方法，会进入call1()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014111626356.png)

调用有参构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241014111738771.png)

关于jackson调用无参构造器和有参构造器的顺序如下；

https://blog.csdn.net/jiayoudangdang/article/details/127813330

- 没有无参构造时：   
  - 如果有参构造的参数全，或者更多(就是有不存在的值)，这样还能正常运行
  - 如果参数不全则直接异常
- 无参构造和有参构造方法都有的时候先走无参构造；   
  - 无参构造需要set/get方法来完成序列化和反序列化


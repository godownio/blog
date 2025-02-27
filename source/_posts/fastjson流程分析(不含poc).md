---
title: "fastjson parseObject分析(纯流程无POC)"
onlyTitle: true
date: 2024-10-6 14:13:23
categories:
- java
- 框架漏洞
tags:
- fastjson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8103.jpg
---

FastJson 是阿里巴巴的开源 JSON 解析库，它可以解析 JSON 格式的字符串，支持 javaBean 序列化为 JSON 字符串，也可以从 JSON 字符串反序列化到 javabean

特点就是快，性能好，也就是因为此，FastJson 使用的用户特别多。但是开源组件+用户大的特点，一旦爆出漏洞，危害也是巨大的

依赖：

```xml
  <dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.24</version>
  </dependency>
    <dependency>
      <groupId>commons-collections</groupId>
      <artifactId>commons-collections</artifactId>
      <version>3.2.1</version>
    </dependency>
```

先打上这个依赖，后面需要换的时候再换

# FastJson解析流程分析

固定数据的fastjson解析：

```java
    public static void main(String[] args) throws Exception{
        String json = "{\"param1\":\"aaa\",\"param2\":\"bbb\"}";
        JSONObject jsonObject = JSON.parseObject(json);
        System.out.println(jsonObject);
    }
```

parseObject把字符串作为json解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241002182315369.png)

也可以给parseObject传入第二个参数，让他按指定的类进行还原，但要求person是一个javabean，如：

```java
Person jsonObject = JSON.parseObject(json,person.class);
```

在json字符串还原成person类的过程中，怎么给字段param1,param2赋值的呢？

答案是调用person类的setter

目前为止代码还挺正常，但是json中加入了一个奇怪的功能，json传入的`@type`能指定以某种类进行还原，如：

```java
        String json = "{\"@type\":\"org.exploit.othercase.Person\",\"param1\":\"aaa\",\"param2\":\"bbb\"}";
```

其功能等效于

```java
    public static void main(String[] args) throws Exception {
        String json = "{\"param1\":\"aaa\",\"param2\":\"bbb\"}";
        Person jsonObject = JSON.parseObject(json, Person.class);
        System.out.println(jsonObject);
    }
```

那看看parseObject究竟干了什么

## 调试

假设有Person这个javabean

```java
package org.exploit.othercase;

//javabean using in fastjsoncase
public class Person {
    public Person(){
        System.out.println("Person Constructor");
    }
    private String param1;
    private String param2;
    public String getParam1() {
        System.out.println("getParam1");
        return param1;
    }
    public void setParam1(String param1) {
        System.out.println("setParam1");
        this.param1 = param1;
    }
    public String getParam2() {
        System.out.println("getParam2");
        return param2;
    }
    public void setParam2(String param2) {
        System.out.println("setParam2");
        this.param2 = param2;
    }
}
```

用@type的方式来还原

```java
        String json = "{\"@type\":\"org.expoilt.othercase.Person\",\"param1\":\"aaa\",\"param2\":\"bbb\"}";
        JSONObject jsonObject = JSON.parseObject(json);
        System.out.println(jsonObject);
```

在输出可以看到，依次调用了构造函数、setter、getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003102916658.png)

酱紫调用？那不是直接能打TemplatesImpl，有CC的话TrAXFilter不是也随便打？

`JSON.parseObject(json);`处打个断点看看怎么个事儿

parseObject调用了parse

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003103253948.png)

在结构中可以看到很多不同参数的parseObject、parse、parseArray。都可能成为以后代审的漏洞入口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003103345930.png)

跟到`JSON.parse(String text,int features)`，实例化了一个DefaultJSONParser

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003105614114.png)

构造函数传入的text（我们的json字符串）赋值为lexer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003105728090.png)

跟到`DefaultJSONParser.parse(Object fieldName)`，里面有许多case

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003104920361.png)

我们传入的是json数据，以`{`开头，进入case LBRACE

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003105208140.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003105245491.png)

跟到DefaultJSONParser.parseObject()

try里的死循环for，就是解析json数据了。简单说一下这里面的逻辑：

* 匹配到逗号`,`跳过
* 匹配两个双引号中间的字符，这里就是`@type`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003125022038.png)

* 如果key为DEFAULT_TYPE_KEY，也就是`@type`，会取后面的value并loadClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003125242734.png)

在这个loadClass里，会判断`[]）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003131336706.png)

然后经典的，双亲委派加载类，forName会进行类的加载和初始化，也就是会执行静态代码块，不会执行构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003135853001.png)

这里就完成了加载Person类。

接着回到DefaulteJSONParser.parseObject，用getDeserializer获取反序列化器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003142925510.png)

跟跟跟，跟到做了一堆判断，选择了创建默认的JavaBean反序列化器。继续跟createJavaBeanDeserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003143539843.png)

asmEnable默认为true，跟进到调用JavaBeanInfo.build

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003160040739.png)

这就是fastjson处理javaBean的核心方法

## javaBeanInfo.build

在这个方法里：

* 先获取了类的所有字段和方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003143847284.png)

* 再获取了默认构造器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003143947126.png)

也就是我们写的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003144043684.png)

* 把构造函数设为可访问

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003144105301.png)

* 共有三个遍历，分别是遍历所有getter，遍历所有public字段，遍历所有getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003180642248.png)

先来看setter的：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003144313276.png)

如果不是setter就跳过，需要满足set开头，参数长度为1，非static，方法名总长度大于3

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003144523824.png)

然后，把set后第一个字母小写，并取后面的字符串为属性名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003144616339.png)

从类中寻找该属性名的字段，如果没有且字段是bool类型，则用`is`加上首字母大写后的字段去查找（所以isName这种也算setter）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003144954173.png)

OK，如果我们本次遍历的方法是setParam1，到这里field就是param1了

把获取到的字段、setter方法名添加到fieldList

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003150216719.png)

这个for循环结束fieldList存在param1和param2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003145917167.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003150324014.png)

* 对getter，如果返回类型是属于Collection 或其子类、Map 或其子类、AtomicBoolean、AtomicInteger、AtomicLong，且没有对应的setter方法，才会把getter加进fieldList

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003153356032.png)

build后有一堆判断，能把asmEnable置为false，目前一个也走不进去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003160415598.png)

如果asmEnable为false，会返回JavaBeanDeserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003162040875.png)

但是这里asmEnable为true，返回asmFactory.creatJavaBeanDeserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003154017421.png)

这个函数创建的FastjsonASMDeserializer_1_Person类似之前学习的临时写在缓存里的代理类，他写了个类然后再去类加载

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003163040027.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003162621852.png)

## 提取FastjsonASMDeserializer_1_Person

可以模拟这个流程把code提出来：

code发布在my github：

https://github.com/godownio/java_unserial_attackcode/blob/master/src/main/java/org/exploit/othercase/fastjson/extract_FastjsonASMDserializer.java

记得改createJavaBeanDeserializer处的输出路径

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003173010126.png)

提取出的FastjsonASMDeserializer如下

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.alibaba.fastjson.parser.deserializer;

import com.alibaba.fastjson.parser.DefaultJSONParser;
import com.alibaba.fastjson.parser.JSONLexerBase;
import com.alibaba.fastjson.parser.ParseContext;
import com.alibaba.fastjson.parser.ParserConfig;
import com.alibaba.fastjson.util.JavaBeanInfo;
import java.lang.reflect.Type;
import org.exploit.othercase.Person;

public class FastjsonASMDeserializer_1_Person extends JavaBeanDeserializer {
    public char[] param2_asm_prefix__ = "\"param2\":".toCharArray();
    public char[] param1_asm_prefix__ = "\"param1\":".toCharArray();
    public ObjectDeserializer param2_asm_deser__;
    public ObjectDeserializer param1_asm_deser__;

    public FastjsonASMDeserializer_1_Person(ParserConfig var1, JavaBeanInfo var2) {
        super(var1, var2);
    }

    public Object createInstance(DefaultJSONParser var1, Type var2) {
        return new Person();
    }

    public Object deserialze(DefaultJSONParser var1, Type var2, Object var3, int var4) {
        JSONLexerBase var5 = (JSONLexerBase)var1.lexer;
        if (var5.token() == 14 && var5.isEnabled(var4, 8192)) {
            return this.deserialzeArrayMapping(var1, var2, var3, (Object)null);
        } else if (var5.isEnabled(512) && var5.scanType("org.exploit.othercase.Person") != -1) {
            Person var8;
            int var12;
            String var14;
            String var15;
            label83: {
                ParseContext var6 = var1.getContext();
                int var7 = 0;
                var8 = new Person();
                ParseContext var9 = var1.getContext();
                ParseContext var10 = var1.setContext(var9, var8, var3);
                if (var5.matchStat != 4) {
                    boolean var11 = false;
                    var12 = 0;
                    boolean var13 = var5.isEnabled(4096);
                    String var10000;
                    if (var13) {
                        var12 |= 1;
                        var10000 = var5.stringDefaultValue();
                    } else {
                        var10000 = null;
                    }

                    var14 = (String)var10000;
                    if (var13) {
                        var12 |= 2;
                        var10000 = var5.stringDefaultValue();
                    } else {
                        var10000 = null;
                    }

                    var15 = (String)var10000;
                    var14 = var5.scanFieldString(this.param1_asm_prefix__);
                    if (var5.matchStat > 0) {
                        var12 |= 1;
                    }

                    if (var5.matchStat == -1) {
                        break label83;
                    }

                    label85: {
                        if (var5.matchStat > 0) {
                            ++var7;
                            if (var5.matchStat == 4) {
                                break label85;
                            }
                        }

                        var15 = var5.scanFieldString(this.param2_asm_prefix__);
                        if (var5.matchStat > 0) {
                            var12 |= 2;
                        }

                        if (var5.matchStat == -1) {
                            break label83;
                        }

                        if (var5.matchStat > 0) {
                            ++var7;
                            if (var5.matchStat == 4) {
                                break label85;
                            }
                        }

                        if (var5.matchStat != 4) {
                            break label83;
                        }
                    }

                    if ((var12 & 1) != 0) {
                        var8.setParam1(var14);
                    }

                    if ((var12 & 2) != 0) {
                        var8.setParam2(var15);
                    }
                }

                var1.setContext(var9);
                if (var10 != null) {
                    var10.object = var8;
                }

                return var8;
            }

            if ((var12 & 1) != 0) {
                var8.setParam1(var14);
            }

            if ((var12 & 2) != 0) {
                var8.setParam2(var15);
            }

            return (Person)this.parseRest(var1, var2, var3, var8, var4);
        } else {
            return super.deserialze(var1, var2, var3, var4);
        }
    }

    public Object deserialzeArrayMapping(DefaultJSONParser var1, Type var2, Object var3, Object var4) {
        JSONLexerBase var9 = (JSONLexerBase)var1.lexer;
        Person var5 = new Person();
        String var6 = var9.scanString(',');
        String var7 = var9.scanString(']');
        var5.setParam2(var7);
        var5.setParam1(var6);
        char var8;
        if ((var8 = var9.getCurrent()) == ',') {
            var9.next();
            var9.setToken(16);
        } else if (var8 == ']') {
            var9.next();
            var9.setToken(15);
        } else if (var8 == 26) {
            var9.next();
            var9.setToken(20);
        } else {
            var9.nextToken(16);
        }

        return var5;
    }
}
```

之后是去调用这个文件里的deserialize。把参数带入，对照文件看

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003173519001.png)

this：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003174827335.png)

clazz就是Person.class，fieldName为null

lexer.token是16，进入else

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003173901258.png)

token=14才进入deserialzeArrayMapping，用于处理数组映射的情况，创建一个新的 Person 对象并设置其字段值。

看到辣吗，在else里:

* new Person()新建了对象，并设置了一些上下文信息
* marchStat根据匹配状态设置标志位，表示哪些字段已被解析
* 根据标志位调用setter方法设置Person对象字段值

return回来的value就是赋好值的Person实例

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003175302656.png)

## another 调试

前面提到，build后有一堆判断，能把asmEnable置为false，但一个也走不进去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003160415598.png)

如果asmEnable为false，会返回JavaBeanDeserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003162040875.png)

据大佬分析，这里特殊手法使asmEnable 走进任意一个if置为false，那就能直接调试了。（但我觉得有可能和前面自然流程生成的字节码不同，就先搞的那个），实际上asmEnable = ture or false都一样。好像是个调试开关

如果你想用这种方法进行调试，需要修改你的Person类：

* 使其一个字段有getter，而没有setter；
* 参数个数不为1
* 返回值是Collection 或其子类、Map 或其子类、AtomicBoolean、AtomicInteger、AtomicLong的一种。

```java
package org.exploit.othercase;

import java.util.Map;

public class intogetOnly_Person {
    public intogetOnly_Person(){
        System.out.println("Person Constructor");
    }
    private Map param1;
    private String param2;
    public Map getParam1() {
        System.out.println("getParam1");
        return param1;
    }
    public String getParam2() {
        System.out.println("getParam2");
        return param2;
    }
    public void setParam2(String param2) {
        System.out.println("setParam2");
        this.param2 = param2;
    }
}
```

```java
    public static void main(String[] args) throws Exception {
//        String json = "{\"@type\":\"org.exploit.othercase.Person\",\"param1\":\"aaa\",\"param2\":\"bbb\"}";
        String json = "{\"@type\":\"org.exploit.othercase.intogetOnly_Person\",\"param2\":\"bbb\"}";
        JSONObject jsonObject = JSON.parseObject(json);
        System.out.println(jsonObject);
    }
```

就会走到JavaBeanInfo.build里把getter添加，跟进一下add实参，`new FieldInfo()`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003180209148.png)

由于参数个数不为1，进入else，把getOnly置为true

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003181518137.png)

就会把asmEnable置为false辣

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241003182145057.png)

来看看这个返回的JavaBeanDeserializer干了啥

额，一堆赋值，标准的构造函数，没别的了。用于后面调用deserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006102636139.png)

ok，我们继续走到JavaBeanDeserializer.deserializer

先调用createInstance进行实例化，触发无参构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006103242942.png)

后面甚至还有获取私有构造函数进行实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006103358051.png)

跟进到fieldDeser.setValue

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006105134353.png)

反射调用setter赋值，over 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006105206876.png)

现在控制台里就触发了构造函数和setter了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006105247434.png)

只是我觉得这种特殊情况进入的调试跟实际情况有些代码可能不同，我觉得还是提取字节码合理一些

那getter什么时候触发的呢？

在parse的后面

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006105417177.png)

## toJSON触发getter

跟进到JSON.getObjectWriter里

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006105725494.png)

又走到createJavaBeanSerializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006110537224.png)

跟进到createASMSerializer，跟前面的setter一样，生成字节码，整笑了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006110746031.png)

提取这个字节码的code发布在：

https://github.com/godownio/java_unserial_attackcode/blob/master/src/main/java/org/exploit/othercase/fastjson/extract_ASMDeserializer.java

提取出的代码如下：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.alibaba.fastjson.serializer;

import java.io.IOException;
import java.lang.reflect.Type;
import org.exploit.othercase.Person;

public class ASMSerializer_1_Person extends JavaBeanSerializer implements ObjectSerializer {
    public ASMSerializer_1_Person(SerializeBeanInfo var1) {
        super(var1);
    }

    public void write(JSONSerializer var1, Object var2, Object var3, Type var4, int var5) throws IOException {
        SerializeWriter var9 = var1.out;
        if (!this.writeDirect(var1)) {
            this.writeNormal(var1, var2, var3, var4, var5);
        } else if (var9.isEnabled(32768)) {
            this.writeDirectNonContext(var1, var2, var3, var4, var5);
        } else {
            Person var10 = (Person)var2;
            if (!this.writeReference(var1, var2, var5)) {
                if (var9.isEnabled(2097152)) {
                    this.writeAsArray(var1, var2, var3, var4, var5);
                } else {
                    SerialContext var11 = var1.getContext();
                    var1.setContext(var11, var2, var3, 0);
                    char var12 = '{';
                    String var6 = "param1";
                    String var13 = var10.getParam1();
                    if (var13 == null) {
                        if (var9.isEnabled(964)) {
                            var9.write(var12);
                            var9.writeFieldNameDirect(var6);
                            var9.writeNull(0, 128);
                            var12 = ',';
                        }
                    } else {
                        var9.writeFieldValueStringWithDoubleQuoteCheck(var12, var6, var13);
                        var12 = ',';
                    }

                    var6 = "param2";
                    var13 = var10.getParam2();
                    if (var13 == null) {
                        if (var9.isEnabled(964)) {
                            var9.write(var12);
                            var9.writeFieldNameDirect(var6);
                            var9.writeNull(0, 128);
                            var12 = ',';
                        }
                    } else {
                        var9.writeFieldValueStringWithDoubleQuoteCheck(var12, var6, var13);
                        var12 = ',';
                    }

                    if (var12 == '{') {
                        var9.write(123);
                    }

                    var9.write(125);
                    var1.setContext(var11);
                }
            }
        }
    }

    public void writeNormal(JSONSerializer var1, Object var2, Object var3, Type var4, int var5) throws IOException {
        SerializeWriter var9 = var1.out;
        if (!var9.isSortField()) {
            this.writeUnsorted(var1, var2, var3, var4, var5);
        } else {
            Person var10 = (Person)var2;
            if (!var9.isEnabled(8192) && !var9.isEnabled(134217728)) {
                if (!this.writeReference(var1, var2, var5)) {
                    if (var9.isEnabled(2097152)) {
                        this.writeAsArrayNormal(var1, var2, var3, var4, var5);
                    } else {
                        SerialContext var11 = var1.getContext();
                        var1.setContext(var11, var2, var3, 0);
                        byte var10000;
                        if (var1.isWriteClassName(var4, var2) && var4 != var2.getClass()) {
                            var9.write(123);
                            this.writeClassName(var1, var2);
                            var10000 = 44;
                        } else {
                            var10000 = 123;
                        }

                        char var12 = (char)var10000;
                        var12 = this.writeBefore(var1, var2, var12);
                        boolean var13 = var9.isNotWriteDefaultValue();
                        var1.checkValue(this);
                        boolean var15 = var1.hasNameFilters(this);
                        String var6 = "param1";
                        Object var8;
                        String var16;
                        if (this.applyName(var1, var2, var6) && this.applyLabel(var1, "")) {
                            var16 = var10.getParam1();
                            if (var13) {
                            }

                            if (this.apply(var1, var2, var6, var16)) {
                                if (var15) {
                                    var6 = this.processKey(var1, var2, var6, var16);
                                }

                                var8 = this.processValue(var1, this.getBeanContext(0), var2, var6, var16);
                                if (var16 != var8) {
                                    if (var8 == null) {
                                        if (var9.isEnabled(964)) {
                                            var9.write(var12);
                                            var9.writeFieldName(var6, false);
                                            var9.writeNull(0, 128);
                                            var12 = ',';
                                        }
                                    } else {
                                        var9.write(var12);
                                        var9.writeFieldName(var6, false);
                                        var1.writeWithFieldName(var8, var6, String.class, 0);
                                        var12 = ',';
                                    }
                                } else if (var16 == null) {
                                    if (var9.isEnabled(964)) {
                                        var9.write(var12);
                                        var9.writeFieldName(var6, false);
                                        var9.writeNull(0, 128);
                                        var12 = ',';
                                    }
                                } else {
                                    var9.writeFieldValue(var12, var6, var16);
                                    var12 = ',';
                                }
                            }
                        }

                        var6 = "param2";
                        if (this.applyName(var1, var2, var6) && this.applyLabel(var1, "")) {
                            var16 = var10.getParam2();
                            if (var13) {
                            }

                            if (this.apply(var1, var2, var6, var16)) {
                                if (var15) {
                                    var6 = this.processKey(var1, var2, var6, var16);
                                }

                                var8 = this.processValue(var1, this.getBeanContext(1), var2, var6, var16);
                                if (var16 != var8) {
                                    if (var8 == null) {
                                        if (var9.isEnabled(964)) {
                                            var9.write(var12);
                                            var9.writeFieldName(var6, false);
                                            var9.writeNull(0, 128);
                                            var12 = ',';
                                        }
                                    } else {
                                        var9.write(var12);
                                        var9.writeFieldName(var6, false);
                                        var1.writeWithFieldName(var8, var6, String.class, 0);
                                        var12 = ',';
                                    }
                                } else if (var16 == null) {
                                    if (var9.isEnabled(964)) {
                                        var9.write(var12);
                                        var9.writeFieldName(var6, false);
                                        var9.writeNull(0, 128);
                                        var12 = ',';
                                    }
                                } else {
                                    var9.writeFieldValue(var12, var6, var16);
                                    var12 = ',';
                                }
                            }
                        }

                        var12 = this.writeAfter(var1, var2, var12);
                        if (var12 == '{') {
                            var9.write(123);
                        }

                        var9.write(125);
                        var1.setContext(var11);
                    }
                }
            } else {
                super.write(var1, var2, var3, var4, var5);
            }
        }
    }

    public void writeDirectNonContext(JSONSerializer var1, Object var2, Object var3, Type var4, int var5) throws IOException {
        SerializeWriter var9 = var1.out;
        Person var10 = (Person)var2;
        if (var9.isEnabled(2097152)) {
            this.writeAsArrayNonContext(var1, var2, var3, var4, var5);
        } else {
            char var11 = '{';
            String var6 = "param1";
            String var12 = var10.getParam1();
            if (var12 == null) {
                if (var9.isEnabled(964)) {
                    var9.write(var11);
                    var9.writeFieldNameDirect(var6);
                    var9.writeNull(0, 128);
                    var11 = ',';
                }
            } else {
                var9.writeFieldValueStringWithDoubleQuoteCheck(var11, var6, var12);
                var11 = ',';
            }

            var6 = "param2";
            var12 = var10.getParam2();
            if (var12 == null) {
                if (var9.isEnabled(964)) {
                    var9.write(var11);
                    var9.writeFieldNameDirect(var6);
                    var9.writeNull(0, 128);
                    var11 = ',';
                }
            } else {
                var9.writeFieldValueStringWithDoubleQuoteCheck(var11, var6, var12);
                var11 = ',';
            }

            if (var11 == '{') {
                var9.write(123);
            }

            var9.write(125);
        }
    }

    public void writeUnsorted(JSONSerializer var1, Object var2, Object var3, Type var4, int var5) throws IOException {
        SerializeWriter var9 = var1.out;
        Person var10 = (Person)var2;
        if (!var9.isEnabled(8192) && !var9.isEnabled(134217728)) {
            if (!this.writeReference(var1, var2, var5)) {
                if (var9.isEnabled(2097152)) {
                    this.writeAsArrayNormal(var1, var2, var3, var4, var5);
                } else {
                    SerialContext var11 = var1.getContext();
                    var1.setContext(var11, var2, var3, 0);
                    byte var10000;
                    if (var1.isWriteClassName(var4, var2) && var4 != var2.getClass()) {
                        var9.write(123);
                        this.writeClassName(var1, var2);
                        var10000 = 44;
                    } else {
                        var10000 = 123;
                    }

                    char var12 = (char)var10000;
                    var12 = this.writeBefore(var1, var2, var12);
                    boolean var13 = var9.isNotWriteDefaultValue();
                    var1.checkValue(this);
                    boolean var15 = var1.hasNameFilters(this);
                    String var6 = "param2";
                    Object var8;
                    String var16;
                    if (this.applyName(var1, var2, var6) && this.applyLabel(var1, "")) {
                        var16 = var10.getParam2();
                        if (var13) {
                        }

                        if (this.apply(var1, var2, var6, var16)) {
                            if (var15) {
                                var6 = this.processKey(var1, var2, var6, var16);
                            }

                            var8 = this.processValue(var1, this.getBeanContext(1), var2, var6, var16);
                            if (var16 != var8) {
                                if (var8 == null) {
                                    if (var9.isEnabled(964)) {
                                        var9.write(var12);
                                        var9.writeFieldName(var6, false);
                                        var9.writeNull(0, 128);
                                        var12 = ',';
                                    }
                                } else {
                                    var9.write(var12);
                                    var9.writeFieldName(var6, false);
                                    var1.writeWithFieldName(var8, var6, String.class, 0);
                                    var12 = ',';
                                }
                            } else if (var16 == null) {
                                if (var9.isEnabled(964)) {
                                    var9.write(var12);
                                    var9.writeFieldName(var6, false);
                                    var9.writeNull(0, 128);
                                    var12 = ',';
                                }
                            } else {
                                var9.writeFieldValue(var12, var6, var16);
                                var12 = ',';
                            }
                        }
                    }

                    var6 = "param1";
                    if (this.applyName(var1, var2, var6) && this.applyLabel(var1, "")) {
                        var16 = var10.getParam1();
                        if (var13) {
                        }

                        if (this.apply(var1, var2, var6, var16)) {
                            if (var15) {
                                var6 = this.processKey(var1, var2, var6, var16);
                            }

                            var8 = this.processValue(var1, this.getBeanContext(0), var2, var6, var16);
                            if (var16 != var8) {
                                if (var8 == null) {
                                    if (var9.isEnabled(964)) {
                                        var9.write(var12);
                                        var9.writeFieldName(var6, false);
                                        var9.writeNull(0, 128);
                                        var12 = ',';
                                    }
                                } else {
                                    var9.write(var12);
                                    var9.writeFieldName(var6, false);
                                    var1.writeWithFieldName(var8, var6, String.class, 0);
                                    var12 = ',';
                                }
                            } else if (var16 == null) {
                                if (var9.isEnabled(964)) {
                                    var9.write(var12);
                                    var9.writeFieldName(var6, false);
                                    var9.writeNull(0, 128);
                                    var12 = ',';
                                }
                            } else {
                                var9.writeFieldValue(var12, var6, var16);
                                var12 = ',';
                            }
                        }
                    }

                    var12 = this.writeAfter(var1, var2, var12);
                    if (var12 == '{') {
                        var9.write(123);
                    }

                    var9.write(125);
                    var1.setContext(var11);
                }
            }
        } else {
            super.write(var1, var2, var3, var4, var5);
        }
    }

    public void writeAsArray(JSONSerializer var1, Object var2, Object var3, Type var4, int var5) throws IOException {
        SerializeWriter var9 = var1.out;
        Person var10 = (Person)var2;
        var9.write(91);
        String var6 = "param1";
        var9.writeString(var10.getParam1(), ',');
        var6 = "param2";
        var9.writeString(var10.getParam2(), ']');
    }

    public void writeAsArrayNormal(JSONSerializer var1, Object var2, Object var3, Type var4, int var5) throws IOException {
        SerializeWriter var9 = var1.out;
        Person var10 = (Person)var2;
        var9.write(91);
        String var6 = "param1";
        var9.writeString(var10.getParam1(), ',');
        var6 = "param2";
        var9.writeString(var10.getParam2(), ']');
    }

    public void writeAsArrayNonContext(JSONSerializer var1, Object var2, Object var3, Type var4, int var5) throws IOException {
        SerializeWriter var9 = var1.out;
        Person var10 = (Person)var2;
        var9.write(91);
        String var6 = "param1";
        var9.writeString(var10.getParam1(), ',');
        var6 = "param2";
        var9.writeString(var10.getParam2(), ']');
    }
}
```

一堆write方法，一眼顶针为序列化Person类。序列化成什么呢？我们后面print结果为json数据，其实就是序列化为json了



该类继承了JavaBeanSerializer，可以强转。并获取了键值对map

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006134116824.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006134231257.png)

这一步之后就打印了调用getter

跟进到getFieldValuesMap内，循环调用了getter.getPropertyValue

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006134516899.png)

接着调用get，且这里fieldInfo就是param1以及对应的getter方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006134816602.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006134854648.png)

反射调用getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006134916547.png)

把获取到的键值对（Entry）循环加入到json数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006135517974.png)

最后得到没有@type的数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006135609133.png)

根据上面的调试，很容易知道，调用不含JSON.toJSON的parse，就不会触发getter，parseArray和一些不同参数的parseObject也不会，具体看具体代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241006135958391.png)

## 总结

fastjson根据@type还原类，是在本地从0开始实例化，然后调用setter赋值

如果解析函数里有JSON.toJSON，还会调用getter

按以下顺序判断，满足条件的话，会被当成setter调用：

* set开头，参数长度为1，非static，方法名总长度大于3
* 没有setter方法，有字段是bool类型，则用`is`加上首字母大写后的字段去查找（所以isName这种也算setter）
* 没有setter方法，有getter方法，参数长度为0，返回类型是属于Collection 或其子类、Map 或其子类、AtomicBoolean、AtomicInteger、AtomicLong的一种


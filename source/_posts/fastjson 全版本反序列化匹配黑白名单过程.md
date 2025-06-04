---
title: "fastjson 全版本反序列化匹配黑白名单过程"
onlyTitle: true
date: 2025-5-27 18:42:06
categories:
- java
- 框架漏洞
tags:
- fastjson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8156.jpg

---



给师姐写了三个论文的密码学代码，燃尽了

回过头来想到上次hvv面试，面试官饶有余味的问我知不知道fastjson 的groovy链，我一想这最新版本的我还真不知道，稀里糊涂的答了应该是AST注解，毕竟groovy在安全除了JNDI 高版本用到的原生类GroovyShell和GroovyClassLoader，就是h2 jdbc attack中用到的AST注解了

现在来学习一下fastjson 中Groovy的利用，但是整个fastjson反序列化的过程忘了，但是这不重要，记得调用构造函数、setter就行了。但更为重要的是几个黑白名单的匹配过程，特来复习和整理一下，把1.2.69也综合进去

## fastjson 全版本反序列化匹配黑白名单过程

不要区分本文的缓存和期望的概念，千万！

本文需要你会fastjson 1.2.68 commons-io链下食用

### fastjson < = 1.2.68

1.2.24<fastjson<1.2.48 利用Class.class 对应 MiscCodec反序列化器，可以提前缓存 在黑名单的类进行绕过

fastjson=1.2.68 虽然新加了一个safeMode安全控制点，如果开启后碰到@type会直接抛出异常，不过safeMode默认是不开启的。在这个版本下可以用expectClass 期望类去绕过。mappings的白名单包括了AutoCloseable和Exception类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024110548113.png)

JavaBeanDeserializer.deserialze和ThrowableDeserializer.deserialze都调用了带期望类参数的checkAutoType

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241024111928928.png)

由于缓存mappings的白名单是Exception，正好Exception是Throwable子类

Throwable子类也是返回ThrowableDeserializer作为反序列化器，那实现Exception就能绕过，不过目前没有实现Exception的库类可以进一步利用，所以暂时跳过ThrowableDeserializer去调用checkAutoType的想法

fastjson 1.2.68 commons-io写文件就是利用的JavaBeanDeserializer 去反序列化 实现了AutoCloseable类的接口（比如ObjectInput 和ObjectOutput）

### fastjson 1.2.69下的反序列化过程

在1.2.69中，ParserConfig的checkAutoType中，若expectClass为AutoCloseable，则设置expectClassFlag为false

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527164413094.png)

而expectClassFlag为true的时候才会直接调用TypeUtils.loadClass了，这导致AutoCloseable为首的利用链都无法使用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527164255929.png)

可能有很多同学很奇怪，为什么会新加一个expectClassFlag去黑名单AutoCloseable，而不是在checkAutotype中把AutoCloseable加到和其他恶意类，比如JdbcRowsetImpl这种类坐一桌的denyHashCodes中。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527164914397.png)

这里注意的是，fastjson在加入checkAutoType机制后的反序列化过程，期望类的黑名单和TypeUtils.loadClass的黑名单不是一个概念。如果一个类在denyHashCodes中，说明不存在反序列化这个类的可能了；而如果一个类在期望类的黑名单中，说明不再能通过调用JavaBeanDesrtializer.deserialze和ThrowableDeserializer.deserialze提前添加该类到期望类中了，后续也就不能利用该类的**子类**进行绕过了，期望类一ban就是ban的该类及其子类。当然还有一些其他差异，遇到看代码就行了，不要钻牛角尖非要区分个场景。

这里列一下fastjson >=1.2.69匹配几个黑白名单的顺序，以免混淆：

* 获取反序列化器阶段：ParserConfig初始化的时候，如果传入的typeName是Class.class，则获取的反序列化构造器为MiscCodec，1.2.48对这里的修复是TypeUtils.loadClass加了一个cache判断，而MiscCodec.deserialze调用时，cache默认为false。直接阻断了cache提前加载恶意类的攻击链路

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241023192305499.png)

* checkAutoType设置expectClass阶段：在设置expectClass阶段，传入的typeName是准备设置的expectClass，传入的expectClass为空（有点绕对吧），因为这里是在设置expectClass。

因为是typeName为待设置的期望类，先从mappings缓存中判断有没有加载过，mappings自带了一个名单，包括AutoCloseable和Exception，这就是期望类的白名单判断过程

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527172925087.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527174539631.png)

比如fastjson 1.2.68 commons-io链，设置期望类时typeName和expectClass如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527181540228.png)

而设置完期望类后对实际的类进行反序列化如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527181653119.png)

也就是说经过了两次checkAutoType才会完成类的loadClass

* checkAutoType反序列化类阶段：先对期望类expectClass进行黑名单匹配（1.2.69黑名单包含AutoCloseable），如果expectClass没有中黑名单，则expectClassFlag为true，进入如下代码，判断要反序列化的类是否在denyHashCodes黑名单中，在的话则抛出异常。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527172425644.png)

如果以上黑名单都没有阻拦到poc的话，则进入反序列化类的阶段。用TypeUtils.loadClass加载类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527182029244.png)

但是我们好像没有看到判断typeName是expectClass子类的阶段呢？比如判断org.apache.commons.io.input.ReaderInputStream是AutoCloseable子类的逻辑，我们怎么没看到？

实际上是在TypeUtils.loadClass完成加载类后，毕竟我们总不能拿着一个字符串的typeName去判断是否为expectClass的子类对吧

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527182548125.png)

我略过了过程中的autoType选项和白名单判断的过程，一切都按默认autoType false ，白名单为空的情况来

这么一看，几个匹配的顺序如下：

根据typeName取出对应的反序列化器 -> 判断expectClass是否在mappings白名单 -> 判断expectClass是否在期望类黑名单 -> 判断需要反序列化的类是否在denyHashCodes黑名单 -> 判断需要反序列化的类是否是expectClass的子类

而实际的poc如何传入expectClass和待反序列化的类？

这里不想分析了，我们可以仿照1.2.68的poc中的一小段，可以看到是用两个同层的@type去传的，第一个是expectClass，第二个是待反序列化的类

```json
{
            "@type": "java.lang.AutoCloseable",
            "@type": "org.apache.commons.io.input.ReaderInputStream",
            "reader": {
                "@type": "org.apache.commons.io.input.CharSequenceReader",
                "charSequence": {
                    "@type": "java.lang.String""aaaaaa...(长度要大于8192，实际写入前8192个字符)"
                },
                "charsetName": "UTF-8",
                "bufferSize": 1024
            }
```


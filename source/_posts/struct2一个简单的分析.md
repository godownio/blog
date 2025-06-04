---
title: "struct2利用"
onlyTitle: true
date: 2023-05-01 13:05:36
categories:
- java
- 框架漏洞
tags:
- struct2
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B870.jpg
---

最近必火出了一个反序列化的视频，出来我就买了，很快啊，马上来篇总结

漏洞环境均为docker compose起的vulhub，然后环境拉出来，IDEA配个远程调试

# struts2

## s2-005

如何快速判断目标主机是否使用了struct2:路径中会存在xxx.action

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230526124630577.png)

依次为执行到下一行，步入和强制步入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230526125956617.png)



如果调试中没有步入，应该是没有导入依赖包。在lib库右键添加库，选择默认(Project library的进行添加



payload：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608095226448.png)

用第一个payload做下调试

`%{"tomcatBinDir{"+@java.lang.System@getProperty("user.dir")+"}"}`，原理就是通过OGNL多次解析%{}，从而调用了java.lang.Sytem的getProperty("user.dir")来获取路径





### 循环解析ognl

strut2 001就是利用了在解析表单时利用了OGNL(%{})来进行解析，入口点在TextParseUtil：

```java
Object result = expression;
```

直接打上断点，记得要右上角下载源码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608092433328.png)

最开始的expression是index.jsp，

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608092655596.png)



一直跳过直到expression为%{username}![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608092812330.png)



开始步过，start来到0

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608093022809.png)

在这里while条件都为真，进入循环，循环就是取%{}中间的值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608093110602.png)



强制跳到end = x - 1

在这里步入可以看到跳到了OgnlValueStack

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608093647089.png)

下面的调试就不详细讲了，一样的思路，算了还是记录一下

一直步过到执行OgnlUtil.getValue()获取表达式的值，步入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608103727072.png)



调用了Ognl对象的方法，继续步入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608103746199.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608103805245.png)

运行到这一步时，reult就是输入的username的值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608103849164.png)



返回到OgnlValueStack，把取出来的username的值赋给了value

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608104019300.png)



上述的步骤就是通过findValue找到username里的值，相当于php里的`value = $_POST['username'];`



因为处于while循环，且每次循环后面都把获取到的值再次赋给了expression。所以实际上这是一个多次解析，也就是%{%{}}嵌套也能进行解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608104126535.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608100558279.png)



我们看第二次循环：这个expression已经是我们第一次取出来的值了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608104200099.png)







再跟一遍，重复的内容略过，看本次解析有什么不一样的：

在OgnlUtil.compile()完成了ognl的解析，且解析出了className和methodName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608114203252.png)



下面讲一下返回ognl子字符串的逻辑（完全可以不看，因为我觉得这里面的三元表达式很有意思）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608114454992.png)



可以看到在这步if判断，tree树已经完成了ognl的取值，所以在跟一下上一步`Object result = ((Node)tree).getValue(ognlContext, root);`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608105226740.png)

在这一步进行强制步入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608104506045.png)

直接就跳到else，然后return了，那跟一下return的方法evaluateGetValueBodys()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608105814199.png)



这里非常容易弄迷糊，先是因为`this.hasConstantValue = false`，直接返回`getValueBody()`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608110029872.png)



getValueBody取到了三个字符串列表result，并把hasConstantValue置为true，于是第二遍的三元表达式return this.constantValue，如此循环完成ognl解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608112930386.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608113001828.png)





得出结论，在OgnlUtil.compile()完成了ognl的解析，且解析出了className和methodName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230608151140892.png)





### k8 exp一把梭

这些老洞都是调着玩的，其实都是工具一把梭。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230617110937910.png)

看路由，为什么是这样回显（问号？）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230617112747978.png)



工具傻🖊都会用，不过多介绍，最多就是看个马的利用方式

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230617111046779.png)



### strut2漏扫

用[](https://github.com/HatBoy)的struts2-scan

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230617112547045.png)

上一步是用s2-005打的，这里扫出来是s2-016，验证一下，怪不得刚才伞兵回显

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230617112832547.png)



## s2-062

漏扫到这没更新了，单独拎出来说

s2-061漏洞环境：https://github.com/Al1ex/CVE-2020-17530

然后改个版本,配个工件和tomcat（我试了从vulhub直接提取代码，结果比用别人的麻烦多了，我的建议是vulhub就别拿来调试了）：

* pom.xml的struts2版本改为2.5.26
* lndexAction里加payload及其get和set方法
* ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230617203336871.png)









### 漏洞分析

s2-062实际上是s2-059和s2-061的延续。你说这struts2从001修到062，一个ognl怎么还没修好



## s2-062漏扫

https://github.com/jax7sec/S2-062

python3 CVE-2021-31805 -u xxxip:port/actionxxx -c "命令"

查看当前权限：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230617164231206.png)













参考文章：

CVE-2021-31805 Apache Struts2 远程代码执行漏洞分析（https://xz.aliyun.com/t/11386）


























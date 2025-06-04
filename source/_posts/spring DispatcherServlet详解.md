---
title: "DispatcherServlet.doDispatch请求分发详解"
onlyTitle: true
date: 2025-3-25 12:49:15
categories:
- java
- java杂谈
tags:
- spring
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8140.png
---



# DispatcherServlet.doDispatch详解

今天来分析一下：

在DispatcherServlet.doDispatch这个至关重要的spring 请求分发函数中

为什么运行了`mappedHandler = DispatcherServlet.getHandler()`，又调用`getHandlerAdapter(mappedHandler.getHandler())`，然后用Adapter.handle去处理请求。这几个Handler究竟负责处理什么？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323221427872.png)



## doDispatch流程

以下列代码举个例子

```java
@Controller
public class TestController {
    @RequestMapping("/index")
    public String index(){
        return "index";
    }
}
```

首先，跟进到第一个getHandler，也就是DispatcherServlet.getHandler()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323222056117.png)

这是一个从handlerMappings中取出handler的方法，取出的handler类型为HandlerExecutionChain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323222204986.png)

handlerMappings内只有两项，分别是RequestMappingHandlerMapping和BeanNameUrlHandlerMapping

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323224006723.png)

跟进到第一次循环，也就是mapping = RequestMappingHandlerMapping时，首先调用getHandlerInternal，然后调用getHandlerExecutionChain，挨着看一下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323224145941.png)

可以看到getHandlerInternal完成了lookupHandlerMethod查找方法，也就是`TestController#index()`了，封装为HandlerMethod后返回

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323224408932.png)

接着跟进getHandlerExecutionChain，先是用HandlerExcutionChain封装handler，然后向这个Chain内添加符合url path的拦截器interceptor

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323224718785.png)

这里返回的结果是我们熟悉的HandlerExecutionChain封装的`TestController#index()` handler。注意这个handler一定是HandlerMethod类型

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323223009554.png)

由此可知：HandlerExecutionChain是添加了interceptor的`TestController#index()`

接着，回到doDispatch，跟进到第二处getHandlerAdapter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323222056117.png)

这里mappedHandler.getHandler是上一步获取的`TestController#index()`  HandlerMethod ，从handlerAdapters中获取了适配该handler的adapter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323223310505.png)

跟进到supports函数内，如果handler是handlerMethod的子类，并且满足supportInternal（恒为true，仅仅测试是否可强转）会返回true。由前文可知，RequestMappingHandlerMapping处理得到的handler一定满足该supports

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323223433493.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323223610161.png)

而handlerAdapaters内只有RequestMappingHandlerAdapter、HttpRequestHandlerAdapter、SimpleControllerHandlerAdapter三个适配器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323223708498.png)

所以一定会取到RequestMappingHandlerAdapter作为HandlerAdapter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323225414691.png)

继续跟进第三处，ha.handle

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323222056117.png)

跟进到了RequestMappingHandlerAdapter父抽象类AbstractHandlerMethodAdapter.handle，调用了handleInternal

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323225604471.png)

跟进到handleInternal，如果开启了session，就验证session调用invokeHandlerMethod去反射调用`TestController#index()`，如果没开启session就直接调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250323225804255.png)

现在算是走完一遍流程

对doDispatch做个总结

1、请求首先进入DispatcherServlet， 由DispatcherServlet 从HandlerMappings中匹配对应的Handler，此时只是获取到了对应的Handler，然后拿着这个Handler去寻找对应的适配器，即：HandlerAdapter；

2、拿到对应HandlerAdapter时，这时候开始调用对应的Handler方法，即执行我们的Controller来处理业务逻辑了， 执行完成之后返回一个ModeAndView；

3、HandlerAdapter执行完之后，返回一个ModeAndView，把它交给我们的视图解析器ViewResolver，通过视图名称查找出对应的视图然后返回；

4、最后，渲染视图 返回渲染后的视图。

但是上面的问题仍然没有得到解答，为什么中间要插入一个HandleAdapter的流程？你会发现去掉这个封装代码仍然可用

## 注册Controller

为什么我上面这么强调一定？

handlerMappings内只有两项，分别是RequestMappingHandlerMapping和BeanNameUrlHandlerMapping

handlerAdapaters内只有RequestMappingHandlerAdapter、HttpRequestHandlerAdapter、SimpleControllerHandlerAdapter三个适配器

BeanNameUrlHandlerMapping、HttpRequestHandlerAdapter、SimpleControllerHandlerAdapter为什么没有用到？

首先，在分析createBean spring的自动装配时提到：

我们分析一下@Controller注解是怎么将类注册进spring的

Spring Controller 本质上是一个 **Spring Bean**。

在 Spring MVC 中，`@Controller` 或 `@RestController` 标注的类会被 Spring 作为 **组件（Bean）** 进行管理。它们通常会被 Spring 的 **组件扫描（Component Scanning）** 机制自动注册为 Spring 容器中的 Bean。

在 Spring 框架中，`@Controller` 本质上是 `@Component` 的一个特殊化注解：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310101443302.png)

其他 Spring 组件（如 `@Service` 和 `@Repository`）也一样，都会被自动扫描并注册到 Spring 容器中。

>Spring 主要依赖以下两种方式进行组件扫描：
>
>- **基于 `@ComponentScan`**
>- **基于 `context:component-scan`（XML 配置）**
>
>其实Spring Boot 应用的 `@SpringBootApplication` 都是 `@ComponentScan` 的一个封装
>
>```java
>@SpringBootApplication  // 等同于 @ComponentScan(basePackages = "com.example")
>public class MyApplication {
>   public static void main(String[] args) {
>       SpringApplication.run(MyApplication.class, args);
>   }
>}
>```
>
>默认情况下，它会扫描 **当前类所在的包及其子包**，所以通常 `SpringBootApplication` 类应该放在 **根包** 位置

但是如果了解过spring的，会记得Bean的依赖注入依靠的是@Autowired注解，那么没有@Autowired注解的spring创建Bean是怎样的？

`AbstractAutowireCapableBeanFactory` 是 Spring Bean 容器中的核心类，它负责 **创建 Bean 并处理依赖注入**。

**Spring 在创建所有 Bean 时，都会经过 `AbstractAutowireCapableBeanFactory` 进行依赖注入，即使该 Bean 没有 `@Autowired` 注解**

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310103843845.png)

所以我们把断点打到AbstractAutowireCapableBeanFactory.createBean，看看怎么把Controller这个bean注册进容器的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310104439827.png)

调用了doCreateBean

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310104525405.png)

在CreateBean内先是创建了RequestMappingHandlerMapping

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310110320763.png)

然后调用了populateBean和initializeBean

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310104819998.png)

跟进到populateBean内，可以看到有根据NAME或者TYPE进行autowired装配

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310105009938.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310105259623.png)

这里我们没有@Autowired标记自动装配的字段，所以populateBean实际上什么都没做就返回了

跟进initializeBean，调用了invokeInitMethods

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310115311253.png)

然后调用bean.afterPropertiesSet，这里bean是上面实例化的RequestMappingHandlerMapping

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310120330264.png)

在RequestMappingHandlerMapping.afterPropertiesSet内，config配置了一堆东西。这里就算把RequestMappingHandlerMapping装配好了。然后调用父类的afterPropertiesSet

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310120419395.png)

调用initHandlerMethods

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310144522272.png)

getCandidateBeanNames获取候选beanName，循环调用processCandidateBean

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310144810781.png)

在processCandidateBean内，如果isHandler(beanType)为true，会调用detectHandlerMethods

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310145015655.png)

跟进到isHandler，就是判断beanType是否用了@Controller注解或者@RequestMapping注解装饰

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310145108104.png)

用了@Controller注解装饰，所以进入detectHandlerMethods

在该方法内，调用getMappingForMethod根据方法和类级别的 @RequestMapping 注解创建 RequestMappingInfo

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310145358853.png)

如果存在类级别的 RequestMappingInfo，则将其与方法级别的合并。

>```java
>@RestController
>@RequestMapping("/api")  // 类级别的映射
>public class MyController {
>
>@RequestMapping("/hello")  // 方法级别的映射
>public String hello() {
>   return "Hello, Spring!";
>}
>}
>```
>
>如上完整路径是 **`/api/hello`**

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310150404557.png)

然后调用MethodIntrospector.selectMethods获取了对应的方法，最后封装如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310150908280.png)

然后是循环调用registerHandlerMethod注册方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310150940307.png)

RequestMappingHandlerMapping.registerHandlerMethod调用了父类的该方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310151109422.png)

AbstractHandlerMethodMapping.registerHandlerMethod调用内部类this.mappingRegistry(AbstractHandlerMethodMapping$MappingRegistry)的register方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310151046492.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310151244604.png)

register把映射的url和对应的方法注册进了AbstractHandlerMethodMapping$MappingRegistry。关键在于装填了mappingLookup和urlLookup。这样在访问index路由时，就能找到需要调用的方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310155833525.png)

下面是已经装配好的registry

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250310155916892.png)

理论上来说，直接调用register就能完成把Controller方法注册进spring内。

## 三种Controller注册方式

也就是说，用@Controller或者@RequestMapping注解的才会用RequestMappingHandlerMapping注册进去。

那么问题来了，创建Controller的方式不止注解一种方式，一共有以下三种方式：

* 方法一：直接使用Controller注解

```java
@Controller
public class TestController {
    @RequestMapping("/index")
    public String index(){
        return "index";
    }
}
```

现在最最最常见的用法，99% Controller都是这么写的

* 方法二：实现Controller接口，实现handleRequest方法。用Component注解进行映射

```java
@Component("/home")
public class HomeController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest request,
                                      HttpServletResponse response) throws Exception {
        System.out.println("home ...");
        response.getWriter().write("home controller from body");
        return null;
    }

}
```

这种方式属于 Spring 早期的 **XML 配置风格**，现在很少使用。

* 方法三：实现HttpRequestHandler接口，实现handleRequest方法。用Component注解进行映射

```java
@Component("/login")
public class LoginController implements HttpRequestHandler {
    @Override
    public void handleRequest(HttpServletRequest request, 
    		HttpServletResponse response) 
    		throws ServletException, IOException {
        System.out.println("login...");
        response.getWriter().write("login ...");
    }
}
```



方法一用RequestMappingHandlerMapping和RequestMappingHandlerAdapter去处理

方法二同样的循环中会进入RequestMappingHandlerMapping

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325122327525.png)

但是因为没有注册的原因，getHandler从mappingLookup和urlLookup中取不到值，于是返回null

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325122351418.png)

于是用`BeanNameUrlHandlerMapping`去实例化，对应的Adapter是SimpleControllerHandlerAdapter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325122758885.png)

方法三用的BeanNameUrlHandlerMapping，对应的Adapter是HttpRequestHandlerAdapter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250325123033486.png)

到这里可以做个总结：

| HandlerMapping               | 使用范围                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| RequestMappingHandlerMapping | 用@Controller或者@RequestMapping注解注册的Controller         |
| BeanNameUrlHandlerMapping    | 实现Controller接口或者实现HttpRequestHandler接口注册的Controller |

| HandlerAdapter                 | 使用范围                                             |
| ------------------------------ | ---------------------------------------------------- |
| RequestMappingHandlerAdapter   | 用@Controller或者@RequestMapping注解注册的Controller |
| SimpleControllerHandlerAdapter | 实现Controller接口                                   |
| HttpRequestHandlerAdapter      | 实现HttpRequestHandler接口                           |

之所以要如此设计，可以观察Controller接口、HttpRequestHandler接口的handleRequest方法。一个返回ModelAndView，一个返回void。说明Controller接口具有视图渲染功能，HttpRequestHandler接口却没有。

正是因为对应的处理不同，所以选择了不同的Adapter去进行相应处理

为什么如此重视这个设计？因为此设计关乎上述的自动视图渲染功能，而此功能在后续分析java ssti漏洞至关重要


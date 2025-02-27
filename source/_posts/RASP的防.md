---
title: "RASP的防"
onlyTitle: true
date: 2024-12-30 10:01:27
categories:
- java
- RASP
tags:
- RASP
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8127.png

---



本文所有代码位于：https://github.com/godownio/RASPAgent

## 环境配置

在讲晦涩难懂的理论之前，先配个代码环境：

https://xz.aliyun.com/t/4902?time__1311=n4%2Bxni0QKmTbG8DBDBqDqpDUO2QooDkbIbReD

https://xz.aliyun.com/t/4903?time__1311=n4%2Bxni0QKmTbG8DyDBqDqpYHQTRZnpoD

按照文1进行环境搭建，文1中文件名应为MANIFEST.MF，文中写错了。文件应有MF配置的图标：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241223215345951.png)

给下我的各项配置：

新建项目后，新建agent模块和test-struts2模块

Module：分别是agent,javawebAgent,test-struts2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241223215408525.png)

目录：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241223215516346.png)

agent下pom.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cn.org.javaweb</groupId>
    <artifactId>agent</artifactId>
    <version>1.0.0</version>
    <name>agent</name>
    <description>Agent Demo</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>org.ow2.asm</groupId>
            <artifactId>asm-all</artifactId>
            <version>5.1</version>
        </dependency>
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.2</version>
        </dependency>
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>

    </dependencies>

    <build>
        <finalName>agent</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.6</source>
                    <target>1.6</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <archive>
                        <manifestFile>src/main/resources/MANIFEST.MF</manifestFile>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <artifactSet>
                                <includes>
                                    <include>commons-io:commons-io:jar:*</include>
                                    <include>org.ow2.asm:asm-all:jar:*</include>
                                </includes>
                            </artifactSet>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.21.0</version>
                <configuration>
                    <skipTests>true</skipTests>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

Tomcat 虚拟机选项：

```
-Dfile.encoding=UTF-8
-noverify
-Xbootclasspath/p:E:\CODE_COLLECT\Idea_java_ProTest\javawebAgent\agent\target\agent.jar
-javaagent:E:\CODE_COLLECT\Idea_java_ProTest\javawebAgent\agent\target\agent.jar
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241223215629562.png)

maven：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241223215643587.png)

按照文1写入类内容

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241223215725059.png)



## RASP初了解

ASM中不同类不同方法的关系图如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/20190427085315-d7534452-6886-1.png)



### code实现

先看code，再了解概念，有时候code比概念的可读性高很多

#### case1

MANIFEST.MF内容（保留最后一个换行）：

```
Manifest-Version: 1.0
Premain-Class: cn.org.javaweb.agent.Agent
Can-Retransform-Classes: true
Can-Redefine-Classes: true
Can-Set-Native-Method-Prefix: true

```

`cn.org.javaweb.agent.Agent`：

```java
import java.lang.instrument.Instrumentation;

public class Agent {

    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new AgentTransform());
    }
}
```

`cn.org.javaweb.agent.AgentTransform`：

```java
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class AgentTransform implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,
                            Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {

        className = className.replace("/", ".");

        System.out.println("Load class:" + className);
        return classfileBuffer;
    }
}
```

运行maven -> 运行Tomcat 后控制台会打印`Load class:xxx`

#### case2

再创建一个`cn.org.javaweb.agent.TestClassVisitor`

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

public class TestClassVisitor extends ClassVisitor implements Opcodes {

    public TestClassVisitor(ClassVisitor cv) {
        super(Opcodes.ASM5, cv);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);

        System.out.println(name + "方法的描述符是：" + desc);
        return mv;
    }
}
```

修改AgentTransform如下：

```java
package cn.org.javaweb.agent;

import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class AgentTransform implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,
                            Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {

        className = className.replace("/", ".");

        try {
            if (className.contains("ProcessBuilder")) {
                System.out.println("Load class: " + className);

                ClassReader classReader  = new ClassReader(classfileBuffer);
                ClassWriter classWriter  = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS);
                ClassVisitor classVisitor = new TestClassVisitor(classWriter);

                classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);

                classfileBuffer = classWriter.toByteArray();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return classfileBuffer;
    }
}
```

首先判断类名是否包含`ProcessBuilder`,如果包含则使用`ClassReader`对字节码进行读取，然后新建一个`ClassWriter`进行对`ClassReader`读取的字节码进行拼接，然后在新建一个我们自定义的`ClassVisitor`类，调用`classReader`的`accept`方法对类的触发事件进行hook，最后给`classWriter`重新赋值修改后的字节码。

TestClassVisitor.transform在哪个地方触发的呢？

在下面触发时序图的MethodVistor前面

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/1569401079000-15689714569539.jpg)

配个Tomcat的cmd.jsp：

模块里新建个web模块

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224104927113.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224105000924.png)

新建一个展开型Web工件，把刚才创建的Web模块搞进去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224105246663.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224105047458.png)

Tomcat部署

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224105259845.png)

目录里现在有了web目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224105402087.png)

向cmd.jsp写入

```jsp
<%@ page import="java.io.InputStream" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<pre>
<%
    Process process = Runtime.getRuntime().exec(request.getParameter("cmd"));
    InputStream in = process.getInputStream();
    int a = 0;
    byte[] b = new byte[1024];

    while ((a = in.read(b)) != -1) {
        System.out.println(new String(b, 0, a));
    }

    in.close();
%>
</pre>
```

Tomcat URL记得改成`ip:port/工件/xxx.jsp`，以便直接打开

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224105436044.png)

访问`http://localhost:8080/agent_Web_exploded/cmd.jsp?cmd=whoami`后，控制台执行打印加载流程和执行结果

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224105537704.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224112227328.png)

case2  TestClassVistor重写ClassVistor.visitMethod ，在AgentTransform匹配到ProcessBuilder时打印`name方法描述符是desc`

那为什么会打印执行命令的所有调用链呢？明明只匹配ProcessBuilder执行了一次

先继续往下看



#### case3

新建一个ProcessBuilderHook类，在类中新建一个start方法

```java
package cn.org.javaweb.agent;

import java.util.Arrays;
import java.util.List;

public class ProcessBuilderHook {

    public static void start(List<String> commands) {
        String[] commandArr = commands.toArray(new String[commands.size()]);
        System.out.println(Arrays.toString(commandArr));
    }
}
```

修改TestClassVisitor类，其中`(Ljava/util/List;)V`定义了一个方法的参数类型和返回类型:

**解析参数列表 `(Ljava/util/List;)`：**

- 参数列表总是用括号 `()` 包裹起来。
- `L` 表示引用类型（对象类型）。
- `Ljava/util/List;` 表示 `java.util.List` 类型的参数，`L` 后面是类的全限定名，末尾用分号 `;` 结尾。
- 括号中的内容表示方法的所有参数类型，这里仅有一个参数，类型为 `java.util.List`。

**解析返回类型 `V`：**

- 返回类型紧跟在括号 `()` 之后。
- `V` 表示方法返回 `void`（无返回值）。

```java
package cn.org.javaweb.agent;

import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;
import org.objectweb.asm.commons.AdviceAdapter;

public class TestClassVisitor extends ClassVisitor implements Opcodes {

    public TestClassVisitor(ClassVisitor cv) {
        super(Opcodes.ASM5, cv);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);

        if ("start".equals(name) && "()Ljava/lang/Process;".equals(desc)) {
            System.out.println(name + "方法的描述符是：" + desc);

            return new AdviceAdapter(Opcodes.ASM5, mv, access, name, desc) {
                @Override
                public void visitCode() {

                    mv.visitVarInsn(ALOAD, 0);
                    mv.visitFieldInsn(GETFIELD, "java/lang/ProcessBuilder", "command", "Ljava/util/List;");
                    mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/agent/ProcessBuilderHook", "start", "(Ljava/util/List;)V", false);

                    super.visitCode();
                }
            };
        }
        return mv;
    }
}
```

在 Java 字节码中，`ALOAD` 指令用于将引用类型的局部变量加载到操作数栈上，而参数 `0` 表示局部变量表中的第 0 个位置。

在 Java 字节码中，每个方法都有一个 **局部变量表**，它是 JVM 的一种数据结构，用来存储方法中的局部变量和方法参数。局部变量表在方法调用时创建，并且只在方法执行期间存活。对于实例方法，局部变量表的第 0 个槽位默认保存当前对象引用 `this`。比如：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229141551069.png)

`ALOAD 0` 就是将 `this` 引用加载到操作数栈



`GETFIELD` 是 Java 字节码中的一条指令，用于从对象中获取实例字段的值。访问 ProcessBuilder 类的 command 字段（类型为 List）

```java
 mv.visitFieldInsn(GETFIELD, "java/lang/ProcessBuilder", "command", "Ljava/util/List;");
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229140543302.png)

`INVOKESTATIC` 是 Java 字节码中的一条指令，用于调用**静态方法**。调用静态方法 cn/org/javaweb/agent/ProcessBuilderHook.start，传递获取的 command 字段;方法签名为 (Ljava/util/List;)V，表示接收一个 List 类型参数且无返回值。

```
mv.visitMethodInsn(INVOKESTATIC, "cn/org/javaweb/agent/ProcessBuilderHook", "start", "(Ljava/util/List;)V", false);
```



上述代码用于执行ProcessBuilder时，打印执行的命令

记住新添加类后需要重新maven打包再运行Tomcat

比如执行python --version，RASP hook到后会打印执行的命令

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224150736940.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241224150749673.png)





我想我不用说，只用细看每个代码，就能体会到RASP是什么，比看概念来的快得多



现在开始碎碎念概念

## RASP概念

这段来自Kodansky Security的Mouad Kondah在INSOMNI HACK的演讲Hijacking The Java Virtual Machine (JVM) And Bypassing Runtime Application Self-Protection (RASP)，B站up 烦星橙做的翻译。

利用Runtime Application Self-Protection (RASP)可以改变调用的软件包功能，和修改库代码权限。RASP是Gartner在2012年推出的一项安全技术。借助RASP，用户不再单纯依赖如WEB应用防火墙（WAF）等外围防护技术。RASP附加到应用程序，部署多个探针以实现全面监控。并实时保护应用程序免受攻击，同时保持较低的误报率和性能开销。

RASP覆盖整个应用堆栈，包括应用程序代码、所使用的库和框架，甚至是商用软件。生成的安全警报可以转发到数据平台进行分析、上传到工单系统或者继承到XDR平台。比如Spring4Shell、Log4Shell和许多其他0day漏洞都可以通过RASP来防止。

市面上的RASP开源解决方案包括OpenRASP、JRasp，商用的包括Contrast SECURITY、imperva、WARATEK、VERACODE、appsealing等。其中大多数是商业解决方案，这也是为何关于这一领域的文献相对较少的原因。

有一些RASP解决方案在HTTP请求由业务代码处理前，经过一系列Filter；还有一些解决方案前置一个Interceptor，一旦检测到恶意请求即刻阻止。但是绕过这些基于流量的检测都十分容易，漏洞利用语言或框架的解析特性混淆或编码字符串进行绕过。这两种解决方案没有利用任何上下文，甚至不如传统的WAF，而且会继承WAF的所有缺点。不仅如此，因为可以看作把WAF融合进了Java程序，一些WAF自身就具有漏洞，在Java环境中被攻击造成的危害更大更广。

大多数解决方案都使用Java Instrumentation API(java agent)来Hook Java进程，而无需修改源代码。唯一采用虚拟化技术方案的是WARATEK，采用宿主机-客户机的方式实现。在JVM虚拟抽象层上运行应用程序，并禁止直接访问Java API类库。改为使用一个纯Java写的虚拟机监控器软件层，该软件层充当了一种权限级别的控制环。但因为全局限制的原因，限制了许多功能的实现。

### RASP的实现概念

首先，执行Instrumentation的agent是使用如ByteBuddy、ASM、Javassist等字节码操作库实现的。当启动Java程序时，主应用程序代码不会立即执行，Java将创建一个JVM，然后加载并启动-javaagent命令行参数所指定的Java agent。而agent会注册一个负责Hook的ClassTransformer，该Transformer用于安全防护。当应用程序正在运行并且正在加载一个新类时，类加载器将在加载此类时在Instrucmentation API上通知ClassTransformer。如果符合Hook规则，类会被修改，称作patched，最后加载到JVM。每当代码与这个类或API交互时，RASP都会察觉，并判断此类代码是否应继续执行。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image6.png)

### RASP、WAF、EDR的区别

RASP、WAF和EDR是三种安全技术，可以单独使用，也可以组合使用，如图\ref{fig7}。WAF一般部署在Web应用程序的前线，作用于流量层，能阻断大部分明显的恶意流量。但是WAF也具有如下缺点：因为WAF严重依赖正则表达式和模式匹配，很容易被绕过；需要大量的人工调整才能应对0day攻击；缺乏上下文，导致较高的误报率。相比之下RASP集成于应用内部，拥有完整的上下文，能在攻击到达主机前阻止并检测攻击。与WAF相比，RASP真正地与应用融为了一体。不仅如此，RASP还对0day攻击有防护能力，且几乎不需要人工调整，导致的也误报很少。最后，EDR在进程级别运行，负责主机安全。EDR获取的信息相对有限，主要管理进程意图、进程链等方面。但在攻击调查期间，明确入侵点至关重要，EDR无法给出攻击入口。

所以RASP在扩展检测与响应(XDR)中发挥着重要作用，提供应用程序层面的监测。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image7.png)

WAF、RASP和EDR并不构成直接竞争关系。在SQL注入的场景，攻击者利用编码技巧绕过了WAF，RASP未能及时patch应用程序时，EDR能在SQL服务器被攻破并试图启动PowerShell脚本时成功阻止攻击。所以WAF、RASP、EDR应该是协同工作，互相补充。

与此同时，Java平台自身就有一个沙箱解决方案，即Java Security Manager(JSM)。JSM通过限制执行调用代码的权限并拒绝访问有价值的资源，如文件系统或网络，来应用最小特权原则。假设攻击者设法加载了一个恶意类，并想要调用ProcessBuilder.start启动一个进程。Security Manager会将权限检查委托给Access Controller，然后Access Controller遍历调用占，确保调用栈上每个调用者都有正确的权限。只要有一个调用者权限不足，就会抛出异常，拒绝访问。

Java Security Manager的问题在于它不具备供应链感知能力，只能管理Java内置类的权限。因此，开发者仍然赋予了应用程序所有库和组件与主代码应用相同的权限，完全违反了零信任原则。攻击者可以绕过内置命令执行类达成漏洞利用，如PostgreSQL外联执行SQL语句。JSM也在JEP 398被移除，证明了这种基于权限的模型的局限性。

相比之下，RASP在设计之初就考虑到了供应链安全，其拥有完整的上下文信息。通过密切观察应用程序的行为和数据流动情况，使得RASP能够更深层次地检查和控制应用程序，超越了JSM所能达到的深度。

### RASP阻断时机

在实现RASP解决方案之前，先要了解Java应用可能遭受的攻击类型及其防护方法。攻击者要入侵应用，首先要找到入口，可能是一个暴露了Web服务的容器，如Netty或Tomcat HTTP。当攻击者发起请求时，应用可能遭受反序列化漏洞、SQL注入等，或包含有可被利用的0day漏洞的第三方库和框架，导致攻击者能在机器上执行任意代码，如图。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image8.png)

一旦攻击者设法在JVM上执行代码，就可以决定是否启动一个进程，EDR会检测到进程启动。但是攻击者可以选择窃取敏感数据，部署恶意库并将其加载到JVM中，不会新启动进程从而逃避EDR的检测。除此以外，攻击者还可以暂时留在JVM中，不留下任何文件的情况下部署一个无文件的WebShell，比如一些通过模拟注册Filter、Interceptor、Controller的Spring Shell，也可以破坏植入应用程序中的所有安全机制，随意越权使用应用的全部功能。

RASP会在应用内部部署大量探针，监控范围不仅涵盖Java API类库，还涵盖正在使用的三方库和Web框架，比如SpringBoot。一旦数据进入应用程序，RASP通过代码插桩（Instrumentation）将其标记为受污染数据。如果RASP在Runtime处进行插桩，之后RASP会追踪数据从请求到Java Runtime库的流动过程。例如攻击者试图从Spring库启动一个进程，或者受污染的数据改变了SQL查询的内容，RASP会对这些典型的攻击模式进行阻止。

RASP通常针对OWASP Top10中大部分注入攻击，如SQL注入、命令注入、JNDI注入等使用黑名单机制。黑名单不仅限于字符串层面，还涉及package层级和gadget层级，如JNDI lookup、文件部署、反序列化代码、表达式语言解释器等。

虽然RASP可以在应用程序不同阶段阻断攻击链，但为了避免绕过，需要尽快阻断攻击。例如，RASP监视到程序试图从Spring启动进程，那么应该立即阻止，而不是等到启动进程后中断进程。因为RASP也是基于Java代码环境，攻击者可以利用一些gadget逃离JVM环境，甚至针对RASP发起攻击。



## 代码层面RASP的实现

### premain

下面的内容来自360 Lucifaer https://paper.seebug.org/1041/

我们以上面的RASP case2举例。

无论用那种模式写出来的agent，都需要将agent打成jar包，同时在jar包中应用`META-INF/MANIFEST.MF`中指定agent的相关信息。这就是我们上面要配置maven打包的原因

```yaml
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: com.lucifaer.javaagentLearning.agent.PreMainTranceAgent
Agent-Class: com.lucifaer.javaagentLearning.agent.AgentMainTranceAgent
```

`Premain-Class`和`Agent-Class`是用来配置不同模式的agent实现类，`Can-Redefine-Classes`和`Can-Retransform-Classes`是用来指示是否允许进行类重定义和类重新转换，这两个参数在一定的情况下决定了是否能在agent中利用ASM对加载的类进行修改。由于我们这里用的premain，所以不要Agent-Class配置也可以

然后是代码实现：

* 需要实现ClassFileTransformer，重载transform方法。当 JVM 加载某个类时，会调用 `transform` 方法，允许开发者对字节码进行修改。

* 需要访问类，所以声明ClassReader，来获取类

* 需要对类中的内容进行修改，所以声明ClassWriter，该类继承于ClassReader

* 实例化访问者classVisitor来进行类访问，所以TestClassVisitor需要继承ClassVisitor，且重载其中的方法来修改字节码：
  - 如果需要访问注解，则实例化`AnnotationVisitor`
  - 如果需要访问参数，则实例化`FieldVisitor`
  - 如果需要访问方法，则实例化`MethodVisitor`

* 最后， *`ClassReader`调用`accept`方法* 完成整个调用流程

```java
package cn.org.javaweb.agent;

import org.objectweb.asm.ClassReader;
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.ClassWriter;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

public class AgentTransform implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,
                            Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {

        className = className.replace("/", ".");

        try {
            if (className.contains("ProcessBuilder")) {
                System.out.println("Load class: " + className);

                ClassReader classReader  = new ClassReader(classfileBuffer);
                ClassWriter classWriter  = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS);
                ClassVisitor classVisitor = new TestClassVisitor(classWriter);

                classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);

                classfileBuffer = classWriter.toByteArray();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return classfileBuffer;
    }
}
```

`ClassFileTransformer` 与 Java Agent 配合使用，通过 `Instrumentation` 提供的 API 注册字节码转换器。

```java
import java.lang.instrument.Instrumentation;

public class Agent {

    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new AgentTransform());
    }
}
```

实例化访问者classVisitor来进行类访问

```java
import org.objectweb.asm.ClassVisitor;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

public class TestClassVisitor extends ClassVisitor implements Opcodes {

    public TestClassVisitor(ClassVisitor cv) {
        super(Opcodes.ASM5, cv);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, desc, signature, exceptions);

        System.out.println(name + "方法的描述符是：" + desc);
        return mv;
    }
}
```

上述说的实例化AnnotationVisitor、FieldVisitor、MethodVisitor分别对应ClassVisitor的visitAnnotation、visitField、visitMethod方法。对应去重载，然后super父类的方法即可。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229120503899.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229120529965.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229120543654.png)

从上面的时序图可以看出修改字节码的顺序，具体代码在ClassReader，太复杂了懒得看。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229120728657.png)



#### 表达式注入监测

下面是一个省略了分离各个组件（流式写法，写到一个文件），监测MVEL，OGNL，SpEL表达式注入漏洞的demo

在 Java 字节码中，`<init>` 是构造方法，构造方法的描述符没有返回值（返回类型始终是 `void`），但它负责将一个类的实例初始化。

```java
package cn.org.javaweb.agent;

import org.objectweb.asm.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.ArrayList;
import java.util.List;

//public class Agent {
//
//    public static void premain(String agentArgs, Instrumentation inst) {
//        inst.addTransformer(new AgentTransform());
//    }
//}

public class Agent implements Opcodes {
    private static List<MethodHookDesc> expClassList = new ArrayList<MethodHookDesc>();

    static {
        expClassList.add(new MethodHookDesc("org.mvel2.MVEL", "eval",
                "(Ljava/lang/String;)Ljava/lang/Object;"));
        expClassList.add(new MethodHookDesc("ognl.Ognl", "parseExpression",
                "(Ljava/lang/String;)Ljava/lang/Object;"));
        expClassList.add(new MethodHookDesc("org.springframework.expression.spel.standard.SpelExpression", "<init>",
                "(Ljava/lang/String;Lorg/springframework/expression/spel/ast/SpelNodeImpl;" +
                        "Lorg/springframework/expression/spel/SpelParserConfiguration;)V"));
    }

    public static void premain(String agentArgs, Instrumentation instrumentation) {
        System.out.println("agentArgs : " + agentArgs);
        instrumentation.addTransformer(new ClassFileTransformer() {
            public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
                final String class_name = className.replace("/", ".");

                for (final MethodHookDesc methodHookDesc : expClassList) {
                    if (methodHookDesc.getHookClassName().equals(class_name)) {
                        final ClassReader classReader = new ClassReader(classfileBuffer);
                        ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS);
                        final int api = ASM5;

                        try {
                            ClassVisitor classVisitor = new ClassVisitor(api, classWriter) {
                                @Override
                                public MethodVisitor visitMethod(int i, String s, String s1, String s2, String[] strings) {
                                    final MethodVisitor methodVisitor = super.visitMethod(i, s, s1, s2, strings);

                                    if (methodHookDesc.getHookMethodName().equals(s) && methodHookDesc.getHookMethodArgTypeDesc().equals(s1)) {
                                        return new MethodVisitor(api, methodVisitor) {
                                            @Override
                                            public void visitCode() {
                                                if ("ognl.Ognl".equals(class_name)||"org.mvel2.MVEL".equals(class_name)) {
                                                    methodVisitor.visitVarInsn(Opcodes.ALOAD, 0);
                                                }else {
                                                    methodVisitor.visitVarInsn(Opcodes.ALOAD, 1);
                                                }
                                                methodVisitor.visitMethodInsn(
                                                        Opcodes.INVOKESTATIC, Agent.class.getName().replace(".", "/"), "expression", "(Ljava/lang/String;)V", false
                                                );
                                            }
                                        };
                                    }
                                    return methodVisitor;
                                }
                            };
                            classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);
                            classfileBuffer = classWriter.toByteArray();
                        }catch (Throwable t) {
                            t.printStackTrace();
                        }
                    }
                }
                return classfileBuffer;
            }
        });
    }

    public static void expression(String exp_demo) {
        System.err.println("---------------------------------EXP-----------------------------------------");
        System.err.println(exp_demo);
        System.err.println("---------------------------------调用链---------------------------------------");

        StackTraceElement[] elements = Thread.currentThread().getStackTrace();

        for (StackTraceElement element : elements) {
            System.err.println(element);
        }

        System.err.println("-----------------------------------------------------------------------------");
    }
}

```

`INVOKESTATIC` 是字节码指令，表示调用一个静态方法。在这里下列代码就是调用本Agent类下的expression静态方法

```java
methodVisitor.visitMethodInsn(Opcodes.INVOKESTATIC, Agent.class.getName().replace(".", "/"), "expression", "(Ljava/lang/String;)V", false);
```

上面 `methodVisitor.visitVarInsn`的结果就作为expression方法的参数。为什么ognl是压栈入局部变量表第0个参数，而其他两个是压栈局部变量表第一个参数？

* 如果是在静态方法中，`0` 指代方法的第一个参数；1指代方法的第二个参数。

- 如果是在实例方法中，索引 `0` 通常指代 `this`，即当前对象；索引 `1` 通常指代方法的第一个参数。

我们锁定到ognl.Ognl#expression的代码，可以看到这是个静态方法，索引为0代表第一个参数，即解析的ognl表达式字符串

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229143307023.png)

方法内定义的局部变量也会按顺序存储在局部变量表中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229143730958.png)

辅助类：

```java
public class MethodHookDesc {
    private String hookClassName;
    private String hookMethodName;
    private String hookMethodArgTypeDesc;

    public MethodHookDesc(String hookClassName, String hookMethodName, String hookMethodArgTypeDesc) {
        this.hookClassName = hookClassName;
        this.hookMethodName = hookMethodName;
        this.hookMethodArgTypeDesc = hookMethodArgTypeDesc;
    }

    public String getHookClassName() {
        return hookClassName;
    }

    public void setHookClassName(String hookClassName) {
        this.hookClassName = hookClassName;
    }

    public String getHookMethodName() {
        return hookMethodName;
    }

    public void setHookMethodName(String hookMethodName) {
        this.hookMethodName = hookMethodName;
    }

    public String getHookMethodArgTypeDesc() {
        return hookMethodArgTypeDesc;
    }

    public void setHookMethodArgTypeDesc(String hookMethodArgTypeDesc) {
        this.hookMethodArgTypeDesc = hookMethodArgTypeDesc;
    }
}
```

打个pom：

```xml
<dependency>
    <groupId>ognl</groupId>
    <artifactId>ognl</artifactId>
    <version>2.7.3</version>
</dependency>
<dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.21.0-GA</version>
</dependency>
```

修改pom中shade的artifactSet：

```xml
<include>ognl:ognl:jar:*</include>
<include>org.javassist:javassist:jar:*</include>
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229161043725.png)

maven打包后重新加个JAR应用程序配置，虚拟机选项依旧得填上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229161109320.png)

MANIFEST.MF加上主类`Main-Class: cn.org.javaweb.agent.MainTest`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229161133286.png)

```java
import ognl.Ognl;
import ognl.OgnlContext;
import ognl.OgnlException;
import org.mvel2.MVEL;

public class MainTest {

    public static void main(String[] args) throws OgnlException {
//        String expression = "new java.lang.ProcessBuilder(\"calc\").start();";
//        MVEL.eval(expression);
        OgnlContext ognlContext = new OgnlContext();

        Ognl.getValue("@java.lang.Runtime@getRuntime().exec('calc')", ognlContext, ognlContext.getRoot());
    }
}
```

运行JAR应用程序后输出了EXP，也就是表达式字符串，还有调用链

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229161229675.png)

org.mvel2.MVEL.eval同理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229163027281.png)

>更深思考一步，假如我监测的是MVELInterpretedRuntime，需要压栈哪个变量才能获取表达式字符串？
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229163603632.png)
>
>继承的AbstractParser如下：
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229163713040.png)
>
>经调试，首先，parse没有接收任何参数，也没有在方法内定义任何变量。即局部变量表只有一个this，其他为空。
>
>方法内使用的stk，dStack是存在哪的呢？数据一定要有地方存储的吧
>
>答案是放在类变量集里的，this引用指向类变量集，属于对象实例。通过this去访问stk，而不是从局部变量表里找。
>
>然后我们又提到static方法局部变量表内没有this，那里面的静态变量怎么访问？
>
>其实被带入误区了，可以直接通过 `类名.静态变量` 访问静态变量，因为静态变量在类的范围内是全局可见的。也不用通过前面的this，笑嘻嘻。
>
>在字节码中，通过指令 `GETSTATIC` 和 `PUTSTATIC` 操作静态变量，而不涉及局部变量表。



然后是SpEL表达书注入

加个SpEL的依赖

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-expression</artifactId>
    <version>5.3.28</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>5.3.28</version>
</dependency>
```

因为是实例方法所以压栈的索引为1，也是成功拦截到了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229165748548.png)



### agentmain

上述demo需要在虚拟机选项，也就是启动参数加上-javaagent，而且每次修改都需要重新打包启动

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229170406090.png)



我们先理解一下agentmain是什么？是运行一个程序时，用一个打包好的agent jar去attach程序，而不是像premain一样合在一起打包成jar

假设我们现在需要attach这个MainTest程序：

```java
package cn.org.javaweb.agent.attach;

import java.io.BufferedReader;
import java.io.InputStreamReader;

public class MainTest {
    public static void main(String[] args) throws Exception{
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
        System.out.println("请输入一个字符串：");
        String input = reader.readLine();
        say(input);
        reader.close();

    }
    public static void say(String str){
        System.out.println("will be validate");
    }

}
```



现在我们制作agentmain 的一个attach jar，两个与premain类似的code：

```java
package cn.org.javaweb.agent.attach.attachjar;

import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class attachAgent {

    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
//        for (Class clazz : inst.getAllLoadedClasses()) {
//            System.out.println(clazz.getName());
//        }
        CustomClassTransformer transformer = new CustomClassTransformer(inst);
        transformer.retransform();
    }
}
```

CustomClassTransformer.java

```java
package cn.org.javaweb.agent.attach.attachjar;

import org.objectweb.asm.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;
import java.security.ProtectionDomain;
import java.util.LinkedList;

public class CustomClassTransformer implements ClassFileTransformer {
    private Instrumentation inst;
    public CustomClassTransformer(Instrumentation inst) {
        this.inst = inst;
        inst.addTransformer(this, true);
    }

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        System.out.println("In Transform");
        ClassReader cr = new ClassReader(classfileBuffer);
        ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_MAXS);
        ClassVisitor cv = new ClassVisitor(Opcodes.ASM5, cw) {
            @Override
            public MethodVisitor visitMethod(int i, final String s, String s1, String s2, String[] strings) {
//                return super.visitMethod(i, s, s1, s2, strings);
                final MethodVisitor mv = super.visitMethod(i, s, s1, s2, strings);
                if ("say".equals(s)) {
                    return new MethodVisitor(Opcodes.ASM5, mv) {
                        @Override
                        public void visitCode() {
                            super.visitCode();
                            mv.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                            mv.visitLdcInsn("CALL " +s+ " method");
                            mv.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
                        }
                    };
                }
                return mv;
            }
        };
        cr.accept(cv, ClassReader.EXPAND_FRAMES);
        classfileBuffer = cw.toByteArray();
        return classfileBuffer;
    }

    public void retransform() throws UnmodifiableClassException {
        LinkedList<Class> retransformClasses = new LinkedList<Class>();
        Class[] loadedClasses = inst.getAllLoadedClasses();
        for (Class clazz : loadedClasses) {
            if ("cn.org.javaweb.agent.attach.MainTest".equals(clazz.getName())) {
                if (inst.isModifiableClass(clazz) && !clazz.getName().startsWith("java.lang.invoke.LambdaForm")) {
                    inst.retransformClasses(clazz);
                }
            }
        }
    }
}
```

类似的部分就不重复解释了。

这里如果方法名为say的话，就使用`GETSTATIC`获取静态变量，对应了System的out变量

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229212810242.png)

修改变量值为`CALL xxx method`，作用就是在调用到say方法时控制台打印这个字符串。

不同的是agentmain需要实现一个额外的函数retransform，使用instumentation.retransformClasses去在运行中修改字节码

1. 获取所有已加载的类。
2. 遍历这些类，查找名为 cn.org.javaweb.agent.attach.MainTest 的类。
3. 检查该类是否可修改且不是 Lambda 表达式相关的类。
4. 如果满足条件，则对该类进行重新转换。

修改MANIFEST.MF：

```yaml
Manifest-Version: 1.0
Agent-Class: cn.org.javaweb.agent.attach.attachjar.attachAgent
Can-Retransform-Classes: true
Can-Redefine-Classes: true
Can-Set-Native-Method-Prefix: true
```

现在maven打包上面修改后的agent模块

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229220737173.png)

由于agentmain是运行中修改，所以还要写个修改的程序：

```java
package cn.org.javaweb.agent.attach;

import com.sun.tools.attach.*;

import java.io.IOException;
import java.util.List;


//JDK>=9
public class Attachit_execinRun {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor vmd : list) {
            if (vmd.displayName().endsWith("MainTest")) {
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                virtualMachine.loadAgent("E:\\CODE_COLLECT\\Idea_java_ProTest\\javawebAgent\\agent\\target\\agent-1.0.0-shaded.jar", "Attach!");
                System.out.println("ok");
                virtualMachine.detach();
            }
        }
    }
}
```

这段代码遍历当前系统中所有的JVM，找到名为"MainTest"结尾的JVM实例，vmd.id()指该JVM的进程号，然后loadAgent去重新Attach

JVM名称是按照现在正在运行的程序来的，你可以把MainTest运行，然后打个断点查看

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229220959088.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229221022947.png)



很有可能你的idea不能识别JDK/lib下的tools.jar，在IDEA->项目结构中库、依赖等导入都无法对maven打包起作用，maven只看pom，那修改pom如下：

```xml
<dependency>
    <groupId>com.sun</groupId>
    <artifactId>tools</artifactId>
    <systemPath>${project.basedir}/lib/tools.jar</systemPath>
    <version>1.8</version>
    <scope>system</scope>
</dependency>
```

模块下新建个lib目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229214105179.png)

直到你的依赖项：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229214052084.png)

下面是个演示：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/RASP%20attach%20InputTest.gif)



我们再次思考"重新"attach的含义，是不是可以重复地attach同一个程序？

答案是对的，下面是attach两次MainTest的结果，可以看到CALL了两次

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229222138886.png)



## question

现在让我们解答case2中提出的问题，为什么会打印命令的所有调用链？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241229223252364.png)

因为当你调用` classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES) `时，ASM 会遍历整个类结构，对于每一个方法，都会调用` TestClassVisitor.visitMethod`（在这里就是ProceeBuilder的每一个方法）。而我们写的代码在触发到ProcessBuilder.start才会调用accept

第二个问题，我们上面的代码都是给方法添加代码，怎么阻断？直接throw SecurityException就OK啦

```java
throw new SecurityException("Detected malicious expression: " + exp_demo);
```

最后一个问题，我们上面的代码都是修改方法，能否添加或删除方法？

如果代码逻辑允许，premain是可以的

但是agentmain不行。从字节码和代码逻辑上来说，利用ASM ACC_PUBLIC可以新建一个public方法

```java
		MethodVisitor mv;
        mv = cw.visitMethod(Opcodes.ACC_PUBLIC, "say2", "()V", null, null);
```

但是agentmain是可以重复attach的，多次attach时，代码会重复插入。原生的JVM在运行时时为了程序的线程及逻辑安全，禁止向运行时的类添加新的public方法并重新定义该类。会报`class redefinition failed: attempted to add a method`错误

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241230095259187.png)



## 总结

以下均来自Lucifaer作者的思考https://paper.seebug.org/1041/

**所有“通用解”的最大问题都出现在通用性上**。在真实场景中RASP的应用环境比其在实验环境中复杂的多，如果想要一个RASP真正的运行在业务上就需要从乙方和甲方的角度双向思考问题

### 语言环境的通配适用性

企业内部的web应用纷繁复杂，有用Java编写的应用，有用Go编写的，还有用PHP、Python写的等等...，那么如何对这些不同语言所构建的应用程序都实现相应的防护？

对于甲方来说，我购置一套安全防护产品肯定是要能起到通用防护的作用的，肯定不会只针对Java购进一套Java RASP，这样做未免也太亏了。

对于乙方来说，每一种语言都有不同的特性，都要用不同的方式构建RASP，对于开发和安全研究人员来说工作量是相当之大的，强如OpenRASP团队目前也只是支持PHP和Java两个版本的。

这很大程度上也是影响到RASP推广的一个原因。看看传统的WAF、旁路流量监测等产品，它并不受语言的限制，只关心流量中是否存在具有威胁的流量就好，巧妙的减少了一个变量，从而加强了泛用性，无论什么样的环境都可以快速部署发挥作用，对于企业来说，肯定是更愿意购入WAF的。

### 部署的通配适用性

由于开发人员所擅长的技能不同或不同项目组的技能树设定的不同，企业内部往往会存在使用各种各样框架实现的代码。而在代码部署上，如果没有一开始就制定严格的规范的话，部署环境也会存在各种各样的情况。就拿Java来说，企业内部可能存在Struts2写的、Spring写的、RichFaces写的等等...，同时这些应用可能部署在不同的中间件上：Tomcat、Weblogic、JBoss、Websphere等等...，不同的框架，不同的中间件部署方式都或多或少的有所不同，想要实现通配，真的不容易。

### 规则的通用性

后面分析OpenRASP会说到，已经被OpenRASP较好的解决了，统一利用js做规则，然后利用js引擎解析规则。

### 自身稳定性的问题

因为RASP是将检测逻辑插入到hook点中的，只要到达了相应的hook点，检测逻辑是一定会被执行的，如果这个时候RASP实现的检测逻辑本身出现了问题，严重的话会导致整个业务崩溃，或直接被打穿。样，如果在RASP所执行的逻辑中出现了严重的错误，将会直接将错误抛出在业务逻辑中，轻则当前业务中断，重则整个服务中断，这对于甲方来说就是严重的事故，甚至比服务器被攻击还严重。

这也就是为什么很多甲方并不喜欢RASP这种方式，因为归根到底，RASP还是将代码插入到业务执行流中，不出问题还好，出了问题就会影响业务。相比来说，WAF最多就是误封，但是并不会down掉业务，稳定性上是有一定保障的。

### 自身安全稳定性

试想一个场景，如果RASP本身存在一定的漏洞，那是不是相当的可怕？即使原来的应用是没有明显的安全威胁的，但是在RASP处理过程中存在漏洞，而恰巧攻击者传入一个利用这样漏洞的payload，将直接在RASP处理流中完成触发。

举个实际的例子，比如在RASP中使用了受漏洞影响的FastJson库来处理相应的json数据，那么当攻击者在发送FastJson反序列化攻击payload的时候就会造成目标系统被RCE。

这其实并不是一个危言耸听的例子，OpenRASP在某版本使用的就是FastJson来处理json字符串，而当时的FastJson版本就是存在漏洞的版本。所以在最新的OpenRASP中，统一使用了较为安全的Gson来处理json字符串。

RASP的处理思路就决定了其与业务是联系非常紧密的，可以说就是业务的“一部分”，所以如果RASP自己的代码不规范不安全，最终将导致直接给业务写了一个漏洞。

### 规则的稳定性

RASP的规则是需要经过专业的安全研究人员反复打磨并且根据业务来定制化的，需要尽量将所有的可能性都考虑进去，同时尽量的减少误报。但是由于规则贡献者水平的参差不齐，很容易导致规则遗漏，从而根本无法拦截相关的攻击，或产生大量的攻击误报。这样对于甲方来说无疑是一笔稳赔的买卖——花费大量时间进行部署，花费大量服务器资源来启用RASP，最终的安全效果却还是不尽如人意。

如果想要尽量的完善规则，只能更加贴近业务场景，针对不同的情况做不同的规则判别。所以说规则和业务场景是分不开的，对乙方来说不深入开发、不深入客户是很难做好安全产品的，如果只是停留在实验阶段，是永远没有办法向工程化和产品化转换的。

### 部署复杂性的问题

不难看理想中最佳的Java RASP实践方式是使用`agentmain`模式进行无侵入部署，但是受限于JVM进程保护机制没有办法对目标类添加新的方法，所以就会造成多次attach造成的重复字节码插入的问题。目前主流的Java RASP推荐的部署方式都是利用`premain`模式进行部署，这就造成了必须停止相关业务，加入相应的启动参数，再开启服务这么一个复杂的过程。

对于甲方来说，重启一次业务完成部署RASP的代价是比较高的，所以都是不愿意采取这样的方案的。而且在甲方企业内部存在那么多的服务，一台台部署显然也是不现实的。目前所提出的自动化部署方案也受限于实际业务场景的复杂性，并不稳定。



就目前来说RASP解决方案已经相对成熟，除非JDK出现新的特性，否则很难出现重大的革新。

目前各家RASP厂商主要都是针对性能及其他的辅助功能进行开发和优化，比如OpenRASP提出了用RASP构建SIEM以及实现被动扫描器的思路，这其实是一个非常好的思路，RASP配合被动扫描器能很方便的对企业内部的资产进行扫描，从而实现一定程度上的漏洞管控。

但是RASP不是万能的，并不能高效的防御所有的漏洞，其优劣势是非常明显的，应当正确的理解RASP本身的司职联合其他的防御措施构建完整的防御体系才能更好的做好安全防护。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241230095914844.png)





参考：

https://paper.seebug.org/1041/

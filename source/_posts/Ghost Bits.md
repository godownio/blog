---
title: "Ghost Bits Bypass WAF"
onlyTitle: true
date: 2026-5-1 20:40:14
categories:
- java
- other漏洞
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8195.jpg
---



## Ghost Bits Bypass WAF

最近太火了，又是浅蓝大神，怎么会这么强

原文：Black Hat Asia 2026 《Cast Attack: A New Threat Posed by Ghost Bits in Java》



### 漏洞原理

其实和overlong UTF-8很像，https://godownio.github.io/2025/06/13/utf-8-overlong-encoding-rao-guo-waf/

overlong UTF-8中，readUTFSpan会按照UTF-8的模板去解析，于是可以把在可见字符的单字节ascii字符强行转为双字节or三字节的UTF码

那么这里的Ghost Bits是什么呢？

其实是开发不规范的原因，用小位数类型去强转大位数类型导致的高位bits被丢弃，但是高位bits会参与UTF字符的形成，导致Bypass WAF，和各种Bypass WAF的姿势一样，这种对于可见字符的构造，不会绕过Rasp

Java 的 `char`类型为 16 位（2 字节），而 `byte`类型仅为 8 位（1 字节）

将 `char`转换为 `byte`时，高 8 位会被丢弃，只保留低 8 位，这一丢失的高位数据即被称为"幽灵比特位（Ghost Bits）"。

依旧是UTF码点代表可见性，从readUTFSpan扩展到了各种强转的场景，额仅此而已

```java
爻  →  U+2F58  →  二进制：00101111 | 00111010
(byte) 转换后：高 8 位 0x2F 丢弃，低 8 位 0x3A → 'X'
```

而网上涉及到这个漏洞就会出现的一个案例：CVE-2025-41242 https://github.com/vulhub/vulhub/pull/773

payload为`阮严灵丰丰甲来/阮严灵丰丰甲来`= `./..` = `../`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260501201247752.png)

Spring Framework 6.2.9下 spring-core/src/main/java/org/springframework/util/StringUtils.java:803存在这种写法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260501201225061.png)

`%u002e`会被解码为`.`，于是`.%u002e` → `..`

overlong UTF-8是会解析整个码点，Ghost Bits会丢弃高位码点，还是有点不一样的。毕竟一个丢弃了数据

### 漏洞场景

如果有这么写代码（其中ch为char）：

```java
(byte) ch
ch & 0xFF
baos.write(ch)
DataOutputStream.writeBytes(...)
```

修复：统一使用 `str.getBytes(StandardCharsets.UTF_8)` 做显式编码转换。

### 可用的漏洞依赖

首先是业务上有propagator是上面这么写的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260501200613215.png)



### 批量

更新你的yakit

一个是fuzz插件，一个fuzz后不太确定是不是对的，可以验证下，还有一个Codec用于单次Ghost bits编码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260501202010296.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260501202328956.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260501202342162.png)

已经猜到有ctf会考Ghost Bits了，那会不会有Ghost Bits WAF+Overlong UTF-8打反序列化的打法呢？

如果是Ghost Bits+Overlong UTF-8，则场景大概率为：将Overlong UTF-8 ObjectStream Ghost Bits编码后作为payload

代码场景为普通WAF后baos.write()后写进文件，再从文件readObject，完美的payload,完美的场景

虽然这样还是会被装上了Ghost Bits的WAF掉，但是想想还是很有趣（可能中间overlong的乱码会被误杀？）



>另外，Burpsuite在发包时也会吞掉unicode中的前8位，导致没法用Burpsuite复现这个漏洞，改用Yakit即可解决。



另外涉及到对字符进制的表达式计算，也会有漏洞，p神的思路这一块，夸张噢

https://t.zsxq.com/3t0Yr
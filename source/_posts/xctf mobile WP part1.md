---
title: "XCTF mobile WP part1"
onlyTitle: true
date: 2025-6-1 17:06:06
categories:
- 安卓逆向
tags:
- mobile
- WP
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8158.png
---



从0开始mobile安全！

学了一下渗透，发现我真的不适合，不喜欢那种不确定性，打了两个HTB，WP写好了我又给删了，我认为我以后不会从事这个工作。我觉得还是能看见代码的东西更适合我

现在从0刷一刷Mobile题，感觉还挺有趣的。边刷边学知识点

IDA有些快捷操作，之后再整理

# XCTF mobile part1

有一说一，现在有了AI，代码审计的工作比渗透之类的好做太多了

## 基础Android

Jadx反编译apk，打开项目后看到MainActivity（这种找main函数的思路在哪都适用）

看到MainActivity，由于是第一次学，解释详细一点

```java
/* loaded from: classes.dex */
public class MainActivity extends AppCompatActivity {
    private Button login;
    private EditText passWord;

    @Override // android.support.v7.app.AppCompatActivity, android.support.v4.app.FragmentActivity, android.support.v4.app.BaseFragmentActivityGingerbread, android.app.Activity
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.acticity_main_1);
        this.passWord = (EditText) findViewById(R.id.passWord);
        this.login = (Button) findViewById(R.id.button);
        this.login.setOnClickListener(new View.OnClickListener() { // from class: com.example.test.ctf02.MainActivity.1
            @Override // android.view.View.OnClickListener
            public void onClick(View v) {
                String str = MainActivity.this.passWord.getText().toString();
                Check check = new Check();
                if (check.checkPassword(str)) {
                    Toast.makeText(MainActivity.this, "Good,Please go on!", 0).show();
                    Intent intent = new Intent(MainActivity.this, (Class<?>) MainActivity2.class);
                    MainActivity.this.startActivity(intent);
                    MainActivity.this.finish();
                    return;
                }
                Toast.makeText(MainActivity.this, "Failed", 0).show();
            }
        });
    }
}
```

`login` 是登录按钮，`passWord` 是用户输入密码的输入框

onCreate方法先加载了 `acticity_main_1.xml` 这个布局文件，然后用findViewById获取界面上的password输入框和登录按钮，最后是最核心的，设置了一个点击事件监听器绑定在登录按钮上

这个监听器通过`MainActivity.this.passWord.getText().toString();`读取密码输入框内容，然后调用check.checkPassword去对输入进行验证，如果校验成功：提示 `"Good,Please go on!"`，跳转到 `MainActivity2`，Intent是创建一个内部窗口

```java
this.login.setOnClickListener(new View.OnClickListener() { // from class: com.example.test.ctf02.MainActivity.1
            @Override // android.view.View.OnClickListener
            public void onClick(View v) {
                String str = MainActivity.this.passWord.getText().toString();
                Check check = new Check();
                if (check.checkPassword(str)) {
                    Toast.makeText(MainActivity.this, "Good,Please go on!", 0).show();
                    Intent intent = new Intent(MainActivity.this, (Class<?>) MainActivity2.class);
                    MainActivity.this.startActivity(intent);
                    MainActivity.this.finish();
                    return;
                }
                Toast.makeText(MainActivity.this, "Failed", 0).show();
            }
        });
```

OK继续看到checkPassword方法，pass的长度为12，重点是要满足下面这个循环

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530112326780.png)

也就是每一位的pass需要满足`(char) (((255 - len) - 100) - pass[len])`==0

我直接写个脚本爆破字符就完了呗

```python
for i in range(12):
    for j in range(32, 127):  # 可打印 ASCII 范围：32（空格）到 126（~）
        # print(f"{i}: {chr(i)}")
        c = chr((255 - i) - 100 - j)
        if c == '0':
            print(f"{chr(j)}",end='')
```

得到kjihgfedcba`

看到MainActivity2，调用了sendBroadcast

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530113749649.png)

sendBroadcast可不是一个自定义函数，而是发送广播，广播内容为一个editText控件的字符串

广播的知识点：

https://www.runoob.com/android/android-broadcast-receivers.html

广播接收器需要实现为BroadcastReceiver类的子类，并重写onReceive()方法来接收以Intent对象为参数的消息。

满足这个要求的是GetAndChange方法，它的onReceive方法只有很短的两句，就是生成NextContent类作为主窗口

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530114412917.png)

这个对象创建时触发onCreate，调用了Init和Change方法，其中Change方法将 APK 中 assets 目录下的 `timg_2.zip` 文件复制到 app 的数据库目录中，命名为 `img.jpg`，然后用它更新界面上的一张图片。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530114552490.png)

值得注意的是，`getApplicationContext().getResources().getAssets()`对应了`assets/`目录

那么我们直接在jdx中导出`assets/time_2.zip`文件，并修改后缀为jpg即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530115049334.png)

图片得到flag

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530115138728.png)

好像我们之前写的脚本没用呢？

我们用mumu打开apk，输入kjihgfedcba`，得到图片如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530115301637.png)

注意到AndroidManifest.xml中配置了一个广播接收器

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530121006475.png)

```
android:exported="true"
```

**允许外部应用** 向这个接收器发送广播（⚠️ 这是个安全关键点）

```
<action>
```

该接收器监听的广播名称（此处是自定义的 `"android.is.very.fun"`）

```java
Intent intent = new Intent("android.is.very.fun");
```

只要你发出的广播 `action` 是 `"android.is.very.fun"`，这个接收器 `GetAndChange` 就会被触发。

所以我们输入android.is.very.fun后就能显示flag

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530121158899.png)

上面导出图片是个非预期，还是按照代码走打好基础

## Android 2.0

经典的算法逆向题，学习IDA的使用！

### 程序逻辑

jadx，开！

看到MainActivity，发现调用了JNI.getResult，如果返回值为1则输出Great

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530133852923.png)

跟进到JNI，发现getResult是个native函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530134044935.png)

注意一下，这种加载动态链接库去调用native方法的，在资源文件一般都会有动态链接库文件，这里是libNative.so

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250530134222595.png)

导出出来，放到IDA分析

在export找到Java_com_example_test_ctf03_JNI_getResult方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531102818891.png)

跳转后F5看到Java_com_example_test_ctf03_JNI_getResult反编译的c代码

```c
bool __fastcall Java_com_example_test_ctf03_JNI_getResult(int a1, int a2, int a3)
{
  int v3; // r4
  const char *v4; // r8
  char *v5; // r6
  char *v6; // r4
  char *v7; // r5
  int i; // r0
  int j; // r0

  v3 = 0;
  v4 = (const char *)(*(int (__fastcall **)(int, int, _DWORD))(*(_DWORD *)a1 + 0x2A4))(a1, a3, 0);
  if ( strlen(v4) == 0xF )
  {
    v5 = (char *)malloc(1u);
    v6 = (char *)malloc(1u);
    v7 = (char *)malloc(1u);
    Init(v5, v6, v7, v4, 0xF);
    if ( !First(v5) )
      return 0;
    for ( i = 0; i != 4; ++i )
      v6[i] ^= v5[i];
    if ( !strcmp(v6, a5) )
    {
      for ( j = 0; j != 4; ++j )
        v7[j] ^= v6[j];
      return strcmp(v7, "AFBo}") == 0;
    }
    else
    {
      return 0;
    }
  }
  return v3;
}
```

简单解释一下这个代码

下面的代码可以简单的表示为`v4 = env->GetStringUTFChars(...)`，也就是说v4是Java层传入的字符串参数

```c
v4 = (*(int (__fastcall **)(int, int, _DWORD))(*(_DWORD *)a1 + 0x2A4))(a1, a3, 0);
```

然后`if (strlen(v4) == 0xF)`检查v4是否为15个字符

然后分配三个缓冲区，调用 `Init(v5, v6, v7, v4, 0xF);`，Init函数肯定给v5,v6,v7分配值了，等下我们再看Init函数的逻辑

接着往后看，对v5进行一个Fisrt判断，如果Fisrt返回false则直接退出程序。

然后是这个循环，将`v6[0..3]` 与 `v5[0..3]` 逐位异或

```c
for ( i = 0; i != 4; ++i )
    v6[i] ^= v5[i];
if ( !strcmp(v6, a5) )
```

比对v6和a5，不过函数前面没看到a5呢？

第二轮异或，将`v6[0..3]` 与 `v7[0..3]`异或，并比较v7与`AFBo}`

```c
for ( j = 0; j != 4; ++j )
    v7[j] ^= v6[j];
return strcmp(v7, "AFBo}") == 0;
```

### 程序分析

看下Init函数在干什么？这种直接甩给gpt看就完了

```c
int __fastcall Init(int result, char *a2, char *a3, const char *a4, int a5)
{
  int v5; // r5
  int v6; // r10
  int v7; // r6

  if ( a5 < 1 )
  {
    v6 = 0;
  }
  else
  {
    v5 = 0;
    v6 = 0;
    do
    {
      v7 = v5 % 3;
      if ( v5 % 3 == 2 )
      {
        a3[v5 / 3u] = a4[v5];
      }
      else if ( v7 == 1 )
      {
        a2[v5 / 3u] = a4[v5];
      }
      else if ( !v7 )
      {
        ++v6;
        *(_BYTE *)(result + v5 / 3u) = a4[v5];
      }
      ++v5;
    }
    while ( a5 != v5 );
  }
  *(_BYTE *)(result + v6) = 0;
  a2[v6] = 0;
  a3[v6] = 0;
  return result;
}
```

根据参数

```c
Init(v5, v6, v7, v4, 0xF);
```

将 15 字符输入 `a4` 拆分成 3 组，每 3 个字符为一个 cycle，第一个放入v5，第二个放入v6，第三个放到v7：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531104643265.png)

继续看下First方法，我们知道First传入的参数是v5，经过``的处理后结果等于`LN^Dl`

```c
bool __fastcall First(char *a1)
{
  int i; // r1

  for ( i = 0; i != 4; ++i )
    a1[i] = (2 * a1[i]) ^ 0x80;
  return strcmp(a1, "LN^dl") == 0;
}
```

```c
    if ( !First(v5) )
      return 0;
```

写一个Python还原v5，还原的公式为`(2 * a1[i]) ^ 0x80`的逆向，`((b ^ 0x80) // 2)`

```python
def inverse_v5():
    target = b"LN^dl"
    res = []
    for b in target:
        orig = ((b ^ 0x80) // 2)
        res.append(orig)
    return bytes(res)

v5_orig = inverse_v5()
print(v5_orig)
```

得到原来的v5为fgorv，注意for循环只修改了四位字符，所以原来的v5应该是fgorl

不过很显然，First是会改变v5的值的，所以后续应该用LN^dl运算

接着，根据下面的代码

```c
for ( i = 0; i != 4; ++i )
      v6[i] ^= v5[i];
    if ( !strcmp(v6, a5) )
```

我们在ida中找一下a5在哪定义的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531111418516.png)

`.rodata` 区段（**只读数据段**）定义了C 风格的字符串数组（`const char a5[]`），起始地址是 `0x00002888`，DCB分配了常量值，

我们关注到字符串的结尾`DCB 0x00      ; null terminator（字符串结尾）`

所以a5的值应该是` 空格5-\x16a`

对应的python脚本解出v6：

```python
def second():
    a5 = [ord(' '),  ord('5'), ord('-'), 0x16, ord('a')]
    # v5 = [ord('f'), ord('g'), ord('o'), ord('r'), ord('l')]
    v5 = [ord('L'), ord('N'), ord('^'), ord('d'), ord('l')]
    v6 = ""
    for i in range(0, 4):
        t = v5[i]^a5[i]
        v6 += (chr(t))
    v6 += 'a'
    print(v6)
second()
```

得到原来的v6为`l{sra`

继续得到v7为`asoy}`

```python
def third():
    a6 = [ord('A'),  ord('F'), ord('B'), ord('o'), ord('}')]
    v6 = [ord(' '),  ord('5'), ord('-'), 0x16, ord('a')]
    v7 = ''
    for i in range(0, 4):
        t = v6[i]^a6[i]
        v7 += (chr(t))
    v7 += '}'
    print(v7)
third()
```

再写一个Init的逆向还原算法，叫gpt写就完了

```python

def reverse_init(v5: bytes, v6: bytes, v7: bytes) -> bytes:
    """
    给定 Init 拆分后的三个等长部分，按 Init 的逻辑还原原始 a4 字符串。
    """
    assert len(v5) == len(v6) == len(v7)
    a4 = bytearray()
    for i in range(len(v5)):
        a4.append(v5[i])  # v5 → 原始位置 % 3 == 0
        a4.append(v6[i])  # v6 → 原始位置 % 3 == 1
        a4.append(v7[i])  # v7 → 原始位置 % 3 == 2
    return bytes(a4)
v5 = b'fgorl'
v6 = b'l{sra'
v7 = b'asoy}'

a4 = reverse_init(v5, v6, v7)
print(a4.decode())  # 可打印出字符串
```

得到flag{sosorryla}



## APK逆向

jadx打开后，核心逻辑就是调用checkSN查看输入的用户名和SN是否合法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531120204798.png)

跟进checkSN，如下

```java
    /* JADX INFO: Access modifiers changed from: private */
    public boolean checkSN(String userName, String sn) {
        if (userName == null) {
            return false;
        }
        try {
            if (userName.length() == 0 || sn == null || sn.length() != 22) {
                return false;
            }
            MessageDigest digest = MessageDigest.getInstance("MD5");
            digest.reset();
            digest.update(userName.getBytes());
            byte[] bytes = digest.digest();
            String hexstr = toHexString(bytes, "");
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < hexstr.length(); i += 2) {
                sb.append(hexstr.charAt(i));
            }
            String userSN = sb.toString();
            return new StringBuilder().append("flag{").append(userSN).append("}").toString().equalsIgnoreCase(sn);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
            return false;
        }
    }

```

对用户名（ `"Tenshine"`）计算 MD5 哈希，然后将将哈希转为十六进制字符串 `hexstr`（32 个十六进制字符）

然后for循环从hexstr中取每隔两个字符的字符，也就是偶数位的字符，最后包装成包装成 `flag{...}`跟用户输入的sn做比对

那就很简单了，直接写个脚本得到sn码：

```python
import hashlib


def generate_sn(user_name: str) -> str:
    md5_bytes = hashlib.md5(user_name.encode()).hexdigest()  # 得到32位hex字符串
    short_md5 = ''.join(md5_bytes[i] for i in range(0, len(md5_bytes), 2))  # 取偶数位字符
    return f"flag{{{short_md5}}}"

# 示例：
print(generate_sn("Tenshine"))
```

flag{bc72f242a6af3857}



### 人民的名义-抓捕赵德汉1-200

jadx打开，在MANIFEST.MF写了入口为CheckPassword

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531142920914.png)

看到CheckPassword，里面有个main方法，调用loadCheckerObject()，然后调用checkerObject.checkPassword去验证输入

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531142855136.png)

跟进到loadCheckObject

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531145258641.png)

从资源路径 `/ClassEnc` 加载加密的 `.class` 文件，AES解密密钥为hexKey，然后用defineClass加载自定义字节码

我们在代码里可以看到hexKey

```java
static String hexKey = "bb27630cf264f8567d185008c10c3f96";
```

资源文件也有ClassEnc

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531143739540.png)

我们直接写个脚本AES解密ClassEnc，然后输出文件，再反编译就能看到ClassEnc

jadx直接导出ClassEnc可能会报错，不过Jar可以直接解压出来的

```python
from Crypto.Cipher import AES

# 你的密钥（十六进制字符串）
hex_key = "bb27630cf264f8567d185008c10c3f96"  # ← 替换成你自己的 hexKey

# 读取加密的ClassEnc文件
with open("ClassEnc", "rb") as f:
    encrypted_data = f.read()

# 转换成字节形式的密钥
key_bytes = bytes.fromhex(hex_key)

# 创建AES解密器（ECB模式）
cipher = AES.new(key_bytes, AES.MODE_ECB)

# 解密
decrypted_data = cipher.decrypt(encrypted_data)

# 保存为class文件
with open("DecryptedClass.class", "wb") as f:
    f.write(decrypted_data)

print("[+] 解密完成，输出文件为：DecryptedClass.class")
```

解密出来ClassEnc如下：

```java
package defpackage;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

/* loaded from: DecryptedClass.class */
public class CheckPass implements CheckInterface {
    public boolean checkPassword(String input) {
        MessageDigest md5Obj = null;
        try {
            md5Obj = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            System.out.println("Hash Algorithm not supported");
            System.exit(-1);
        }
        byte[] bArr = new byte[40];
        md5Obj.update(input.getBytes(), 0, input.length());
        byte[] hashBytes = md5Obj.digest();
        return byteArrayToHexString(hashBytes).equals("fa3733c647dca53a66cf8df953c2d539");
    }

    private static String byteArrayToHexString(byte[] data) {
        int i;
        StringBuffer buf = new StringBuffer();
        for (int i2 = 0; i2 < data.length; i2++) {
            int halfbyte = (data[i2] >>> 4) & 15;
            int two_halfs = 0;
            do {
                if (halfbyte >= 0 && halfbyte <= 9) {
                    buf.append((char) (48 + halfbyte));
                } else {
                    buf.append((char) (97 + (halfbyte - 10)));
                }
                halfbyte = data[i2] & 15;
                i = two_halfs;
                two_halfs++;
            } while (i < 1);
        }
        return buf.toString();
    }
}
```

checkPassWord要求对输入进行MD5后等于"fa3733c647dca53a66cf8df953c2d539"，直接上cmd5解密得到monkey99

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531145040541.png)

flag{monkey99}



## ill-intentions

进来看到MainActivity，创建一个 `IntentFilter`，监听广播事件 `com.ctf.INCOMING_INTENT`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531151551286.png)

注册`Send_to_Activity` 的自定义广播接收器。并指定 **接收广播需要有权限 `Manifest.permission._MSG`**。

跟进到Send_toActivity，根据不同的msg字符串，分别调用ThisIsTheRealOne、IsThisTheRealOne、DefinitelyNotThisOne

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531170115327.png)

简单的看一下ThisIsTheRealOne类

构建了一个abc组合的msg intent进行广播，这里好几个native方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531171632906.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531171507958.png)

类中声明了几个 native 方法（JNI），并在 `static` 块加载了 `hello-jni` 库

创建一个 `Intent`，设置 Action 为自定义广播 `"com.ctf.OUTGOING_INTENT"`

```java
Intent intent = new Intent();
intent.setAction("com.ctf.OUTGOING_INTENT");
```

这几行是构造 JNI 函数的三个参数：

- `a` 是字符串资源 `str2` 加 `"YSmks"`。
- `b` 和 `c` 是通过 `Utilities.doBoth(...)` 处理两个字符串：
  - 一个是 `R.string.dev_name`
  - 另一个是当前类名

```java
String a = getResources().getString(R.string.str2) + "YSmks";
String b = Utilities.doBoth(getResources().getString(R.string.dev_name));
String c = Utilities.doBoth(getClass().getName());
```

然后调用native方法 `orThat(String, String, String)`，结果放入 `Intent` 的 `"msg"` 字段中。

```java
intent.putExtra("msg", ThisIsTheRealOne.this.orThat(a, b, c));
```

发送广播 `com.ctf.OUTGOING_INTENT`，附带权限 `Manifest.permission._MSG`，表明接收者必须声明该权限才能收到该广播。

```java
sendBroadcast(intent, Manifest.permission._MSG);
```





### Frida环境搭建

播段小插曲，这个题需要用到frida，这里装一下，我测试17.x有bug

```shell
pip install frida==16.7.19
pip install frida-tools==13.7.1
pip install objection
```

然后下对应版本的frida-server

https://www.python.org/downloads/windows/

怎么传上去运行呢？用到了Android  设备和计算机之间通信的命令行工具，叫adb

adb链接mumu：https://blog.csdn.net/qq_51250393/article/details/133980958

如下代表链接上了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531180612504.png)

查看系统版本

```shell
adb shell getprop ro.product.cpu.abi
```

解压后通过adb push将frida server推到设备临时目录

```sh
adb push frida-server-xxx /data/local/tmp/
```

`adb shell`进入设备shell环境，cd到临时目录下，这里需要以root权限运行frida server，先mumu模拟器把手机root选项开启

https://blog.csdn.net/redrose2100/article/details/129481321

然后adb shell里面`su`，就会弹出以下窗口，同意即可

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531181536923.png)

给frida-server文件设置可执行权限使其可以运行

```shell
chmod 777 frida-server-xx
./frida-server-xx
```

这样就算运行成功了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250601115229061.png)

如果显示Aborted！那是你的frida server版本装错了

之后需要转发frida的服务端口，之前运行的frida-server窗口别关，新开一个cmd窗口

```shell
PS C:\Users\19583> adb forward tcp:27042 tcp:27042
27042
PS C:\Users\19583> adb forward tcp:27043 tcp:27043
27043
# 列出设备
PS C:\Users\19583> frida-ls-devices
Id              Type    Name             OS
--------------  ------  ---------------  ------------------
local           local   Local System     Windows 10.0.26100
127.0.0.1:7555  usb     SM S9110
barebone        remote  GDB Remote Stub
socket          remote  Local Socket


# 列出进程
## -U 连接usb设备
PS C:\Users\19583> frida-ps -U
 PID  Name
----  ------------------------------------------------
1142  adbd
1810  android.ext.services
1125  android.hardware.audio.service
1126  android.hardware.bluetooth@1.1-service.sim
1127  android.hardware.camera.provider@2.4-service
1128  android.hardware.configstore@1.1-service
...
```



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531184723779.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250531185246564.png)

能运行说明OK了



### Frida Hook题解

回到题目，一共三个类，如果静态分析，需要把三个类往msg放的orThat输出的字符串分析出来

但是我们顺其自然，直接想办法去触发这几个if条件，不就能获取到结果了吗

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250601103357087.png)

甚至我还有一个思路，把代码copy出来重新跑一遍就能得到结果，不过这学不到什么技术，在此略过

这里学一下Frida Objection动态Hook的思路

Frida注入实际上写的是JS脚本

#### Frida objection的使用

首先`frida-ps -Uai`看下要注入的包名：为com.example.hellojni

```sh
C:\Users\19583>frida-ps -Uai
 PID  Name             Identifier
----  ---------------  -----------------------
5830  CTF Application  com.example.hellojni
3549  文件               com.android.documentsui
2303  游戏中心             com.mumu.store
4633  设置               com.android.settings
   -  主题装扮             com.mumu.decorate
   -  图库               com.android.gallery3d
   -  应用分身             com.netease.mumu.cloner
   -  浏览器              com.android.browser
   -  相机               com.android.camera2
   -  虚拟身份管理           com.nemu.oaidmanager
```

已注入Frida的情况下（客户端执行frida-server）使用objection，因为server已经链接上了client，所以直接在本机执行objection就行了

```sh
objection -g com.example.hellojni explore
android hooking list activities
```

* `objection -g com.example.hellojni explore`使用 `objection` 工具连接到一个目标 Android 应用，包名为 `com.example.hellojni`

  `-g` 指定目标应用的包名

  `explore` 进入交互式 Objection shell（REPL），可以进行 Frida hook、dump、Intent 启动等操作

android hooking list activities可以看到有四个Activity

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250601150921282.png)

随便启动一个Activity

```sh
android intent launch_activity com.example.application.IsThisTheRealOne
```

`android intent launch_activity com.example.application.IsThisTheRealOne`用于尝试启动一个 Activity

`android intent launch_activity`：命令告诉 Objection 构造一个 `Intent` 并使用 `startActivity()` 来启动目标界面

`com.example.application.IsThisTheRealOne`：是你想要启动的 Activity 的类全名（包名+类名）

实际上等于adb的：`am start -n com.example.hellojni/com.example.application.IsThisTheRealOne`

因为代码逻辑中，点击后实际上是调用一个sendBroadcast发送广播，所以并没有任何消息弹出，这个时候需要写一个JS去Hook Activity的结果

比如ThisIsTheRealOne，实际上程序运行的结果为orThat，我们可以直接用objection提供的demo改一下进行Hook

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250601151716577.png)

如下代码，就是hook了orThat，在函数开始和函数结束插入代码分别打印入参和return值

```javascript
var ThisIsTheRealOneHandler = Java.use('com.example.application.ThisIsTheRealOne')
        ThisIsTheRealOneHandler.orThat.overload('java.lang.String', 'java.lang.String', 'java.lang.String').implementation = function(arg0, arg1, arg2) {
            console.log('进入 ThisIsTheRealOneHandler 函数，参数1: ' + arg0 + " 参数2: " + arg1 + "  参数3: " + arg2)
            var ret =  this.orThat(arg0, arg1, arg2)
            console.log('完成 ThisIsTheRealOneHandler 函数，返回值：' + ret )
            return ret
        }
```

我们这里调用了`Java.use('com.example.application.ThisIsTheRealOne')`的时候，就会触发ThisIsTheRealOne的static块，也就实现了so的加载

这是一个通用的demo，以后写的frida也能从这基础上改

类似于java agentmain，frida实现的插桩可以位于native层，所以切面会广得多

类似的，去hook DefinitelyNotThisOne definitelyNotThis方法和IsThisTheRealOne perhapsThis方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250601152703484.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250601152800722.png)

```javascript
function main() {
    Java.perform(function() {
        var DefinitelyNotThisOneHandler = Java.use('com.example.application.DefinitelyNotThisOne')
        DefinitelyNotThisOneHandler.definitelyNotThis.implementation = function(arg0, arg1) {
            console.log('进入 DefinitelyNotThisOneHandler 函数，参数1: ' + arg0 + " 参数2: " + arg1)
            var ret = this.definitelyNotThis(arg0, arg1)
            console.log('完成 DefinitelyNotThisOneHandler 函数，返回值: ' + ret )
            return ret
        }

        var ThisIsTheRealOneHandler = Java.use('com.example.application.ThisIsTheRealOne')
        ThisIsTheRealOneHandler.orThat.overload('java.lang.String', 'java.lang.String', 'java.lang.String').implementation = function(arg0, arg1, arg2) {
            console.log('进入 ThisIsTheRealOneHandler 函数，参数1: ' + arg0 + " 参数2: " + arg1 + "  参数3: " + arg2)
            var ret =  this.orThat(arg0, arg1, arg2)
            console.log('完成 ThisIsTheRealOneHandler 函数，返回值：' + ret )
            return ret
        }

        var IsThisTheRealOneHandler = Java.use('com.example.application.IsThisTheRealOne')
        IsThisTheRealOneHandler.perhapsThis.overload('java.lang.String', 'java.lang.String', 'java.lang.String').implementation = function(arg0, arg1, arg2) {
            console.log('进入 IsThisTheRealOneHandler 函数，参数1: ' + arg0 + "  参数2: " + arg1 + "  参数3: " + arg2)
            var ret =  this.perhapsThis(arg0, arg1, arg2)
            console.log('完成 IsThisTheRealOneHandler 函数，返回值： ' + ret )
            return ret
        }
    })
}

setImmediate(main)
```

保存为agent.js，然后用frida去hook

```sh
frida -U -N com.example.hellojni -l agent.js
```

这个时候再利用objection进入到Activity，点击按钮后frida控制台就会打印相关内容

```sh
PS C:\Users\19583> objection -d -g com.example.hellojni explore
...
Agent injected and responds ok!

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.10.2

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
com.example.hellojni on (Samsung: 12) [usb] # android intent launch_activity com.example.application.IsThisTheRealOne
```

IsThisTheRealOne得到flag CTF{IDontHaveABadjokeSorry}

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250601164457097.png)




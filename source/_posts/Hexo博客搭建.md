---
title: "Hexo博客搭建"
onlyTitle: true
date: 2021-11-28 13:05:36
categories:
- Hexo
tags:
- hexo
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B870.jpg
---

去掉图片图注正则表达式：`!\[[^\]]*\]`,上次处理到2024ciscn wp
# Problem
1. hexo d老是卡住：`git config --global http.proxy 'socks5://127.0.0.1:10086'`
    和`git config --global https.proxy 'socks5://127.0.0.1:10086'`开代理
2. hexo d后需要等一两分钟github渲染
3. 生成密钥`ssh-keygen -t rsa -C "1958304602@qq.com"`，然后一路回车，默认就行了

4. 图片压缩软件：Caesium

5. 报错`Template render error: (unknown path)` ，hexo转义时候发生的错误，你文章中可能出现了`{{}}`，`{% %}`这种hexo无法转义的字符


## commit了但是workflow一直在deploy

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230629135406086.png)

直接把github pages下线：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230629135518254.png)

找最近一次部署成功的页面再发布一次

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230629135544231.png)

****

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230629135618866.png)

再hexo clean ,hexo d就更新成功了。

问题的原因应该是这次发布成功之后最近一次发布，里面写的语法有错误，比如图片没有上传用的本地图片等。


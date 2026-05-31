---
title: "Dynamic Taint Tracking for Modern Java Virtual Machines"
onlyTitle: true
date: 2025-11-25 16:34:32
categories:
- 杂七杂八
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8180.png

---



### [Dynamic Taint Tracking for Modern Java Virtual Machines](https://dl.acm.org/doi/abs/10.1145/3729349)

科研🦆🍐很大，2025 ACM顶刊污点分析拜读

## 影子存储与镜像存储

镜像存储：把变量导入到另一个函数模拟执行操作

>**不直接修改**原始程序的栈帧、方法签名或类结构。
>
>通过一个**独立的、并行的系统**（例如，在另一个线程或一个全局的数据结构中）来“镜像”或映射原始程序的状态，并在这个独立的系统中维护和传播污点标签。
>
>- **优点：侵入性弱，更稳健**
>  - 因为对原始代码的修改较少，所以应用程序的正常行为受影响较小，更不容易崩溃，更加稳健。
>- **缺点：可能不够精确**
>  - 由于污点跟踪逻辑与程序主逻辑是“分离”的，它可能无法完美地捕捉到所有细微的运行时状态，导致污点传播的精确性下降（例如，在某些复杂控制流下可能丢失标签）。

镜像存储的插桩后伪代码

```java
// 全局的污点映射表
Map<Object, Boolean> taintMap = new HashMap<>();

public int processData(int sensitive, int normal) {
    // 在独立的"镜像"系统中记录操作
    TaintTracker.recordMethodEntry("processData");
    
    // 记录参数污点状态
    TaintTracker.recordParameter(0, sensitive); // 参数0 = sensitive
    TaintTracker.recordParameter(1, normal);    // 参数1 = normal
    
    // 执行原始操作
    int result = sensitive + normal;
    
    // 在镜像系统中执行污点传播
    TaintTracker.recordBinaryOperation("IADD", sensitive, normal, result);
    
    // 根据污点规则计算结果的污点状态
    boolean s_sensitive = TaintTracker.getTaint(sensitive);
    boolean s_normal = TaintTracker.getTaint(normal);
    boolean s_result = s_sensitive || s_normal;
    
    // 在镜像中存储结果的污点标签
    TaintTracker.setTaint(result, s_result);
    
    TaintTracker.recordMethodExit("processData", result);
    return result;
}

// TaintTracker 类的简化实现
class TaintTracker {
    static Map<Object, Boolean> valueTaints = new HashMap<>();
    static Stack<OperationContext> contextStack = new Stack<>();
    
    static void recordMethodEntry(String methodName) {
        contextStack.push(new OperationContext(methodName));
    }
    
    static void recordParameter(int index, Object value) {
        // 记录参数值与其污点状态的映射
    }
    
    static void recordBinaryOperation(String op, Object left, Object right, Object result) {
        boolean taintLeft = getTaint(left);
        boolean taintRight = getTaint(right);
        boolean taintResult = taintLeft || taintRight; // 传播规则
        setTaint(result, taintResult);
    }
    
    static boolean getTaint(Object value) {
        return valueTaints.getOrDefault(value, false);
    }
    
    static void setTaint(Object value, boolean taint) {
        valueTaints.put(value, taint);
    }
}
```



影子存储：为每一个原始数据都创建一个并行的“影子”来存储其污点标签。dicksuck描述：

>#### 1. 如何存储标签？
>
>- **局部变量**：在同一个栈帧中，为每个原始局部变量创建一个新的“影子局部变量”来存储其污点标签。
>- **操作数栈**：修改操作数栈，使得每个原始操作数旁边都紧挨着存储其对应的“影子操作数”（即标签）。
>- **字段**：为每个类的原始字段添加一个“影子字段”。
>- **数组**：用一个新的包装对象替换原始数组，这个包装对象内部既存储原始数组，也存储每个数组元素和数组长度的污点标签。
>
>#### 2. 如何传递标签？
>
>- 为每个原始方法创建一个对应的“影子方法”。
>- 影子方法的参数列表是原始参数列表加上对应的“影子参数”（用于传入每个参数的污点标签）。
>- 如果方法有返回值，还会额外添加一个影子参数，用于**传出**返回值的污点标签。
>- 程序中的所有方法调用都被替换为对相应影子方法的调用。
>
>#### 3. 如何传播标签？
>
>- 将计算和传播污点标签的逻辑**直接插入**到原始方法体的字节码中。
>- 标签的运算和原始程序的逻辑在**同一个方法调用、同一个栈帧**中执行。
>
>#### 影子存储的优缺点
>
>- **优点：精确性高**
>  - 因为污点数据的存储和运算与原始程序保持完全同步（在同一个栈帧、同一逻辑流中），所以它能非常精确地模拟 JVM 的运行时语义，污点传播的结果很准确。
>- **缺点：侵入性强，可能导致问题**
>  - **“侵入性”** 指的是它对原始程序的字节码修改非常深、范围非常广（改变了栈结构、方法签名、类结构等）。
>  - 这种深度的修改可能导致应用程序的**行为与正常运行时不一致**，比如引发意外的崩溃或兼容性问题。文中提到，其他研究和他们自己的评估都观察到了这种现象。

影子存储插桩后的伪代码：

```java
// 影子方法 - 添加了影子参数来传递污点标签
public int processData_shadow(
    int sensitive,       // 原始参数
    boolean s_sensitive, // sensitive的影子参数（污点标签）
    int normal,          // 原始参数  
    boolean s_normal,    // normal的影子参数（污点标签）
    boolean[] s_return   // 返回值的影子参数（传出）
) {
    // 影子局部变量
    boolean s_result;
    
    // 加载原始值和对应的影子标签
    // LOAD sensitive, s_sensitive
    // LOAD normal, s_normal
    
    // 执行原始操作：加法
    int result = sensitive + normal;
    
    // 污点传播逻辑：如果任一操作数被污染，结果也被污染
    s_result = s_sensitive || s_normal;
    
    // 存储返回值的污点标签
    s_return[0] = s_result;
    
    return result;
}

// 调用处的插桩
// 原始调用：processData(x, y)
// 变为：
boolean[] return_taint = new boolean[1];
int result = processData_shadow(x, taint_x, y, taint_y, return_taint);
boolean result_taint = return_taint[0];
```



- **影子存储的“长”**：**精确性**。因为污点标签的存储和传播与原始程序逻辑在同一个栈帧、同一方法体中同步进行，能完美匹配JVM语义。
- **影子存储的“短”**：**侵入性过强**。修改方法签名、用包装器包裹数组等操作，极易导致程序行为异常或崩溃，尤其是在面对现代Java特性（如模块系统、原生方法）时。
- **镜像存储的“长”**：**稳健性**。不改变原始程序的结构，只是在一个平行的“镜像空间”中模拟执行，因此对程序正常行为影响小。
- **镜像存储的“短”**：**不精确**。由于污点逻辑与程序逻辑分离，难以精确模拟所有JVM语义，且容易因JVM的“上行调用”导致污点标签传递错位。



## Gelette设计架构

### 操作数栈的处理

采用影子存储，但是从紧挨的影子栈变成对称影子栈

传统影子存储会修改操作数栈的结构，使其每个原始值下面紧挨着存储其标签。这非常复杂且容易出错（例如，处理`DUP2_X2`这样的指令时会生成大量代码，可能超出方法体积限制）。

Galette将影子操作数栈存储在方法的局部变量表中。

它将局部变量表的后半部分开辟为“影子栈”的空间。这后半部分就作为栈，影子栈的“栈顶”始终位于局部变量表的最后一个索引位置。当原始程序对操作数栈进行压栈、出栈等操作时，Galette的插桩代码会在影子栈的对应区域执行完全相同的操作，只不过操作的对象是污点标签。

原来需要顺序执行的栈操作，现在对称执行就行了

### 字段处理

采用传统影子存储

为每个原始字段添加一个对应的“影子字段”（例如，字段`count`对应的影子字段是`count$$Tag`）。

影子字段与原始字段在同一个类中声明，具有相同的访问修饰符。这使得JVM在解析字段访问时，其语义与原始程序完全一致，无需像镜像存储那样去模拟字段解析过程，从而保证了精确性。

### 数组处理

采用镜像存储

传统影子存储会用一个新的“包装器对象”替换原始数组，这会带来严重问题：

- 当包装器数组被传递给未插桩的代码（尤其是**原生方法**）时，对方期望的是原始数组，收到包装器会导致崩溃或未定义行为。（说人话就是类型不匹配，明明要的是ArrayList，你非要封装一个MyHashMap给我）

Galette的解决方案：维护一个全局的、类似于镜像存储中“镜像堆”的映射结构——`TagStore`。

- `TagStore`将一个原始数组对象映射到一个`ArrayTags`记录，该记录包含了该数组长度和每个元素的污点标签。
- 当访问数组元素时，插桩代码会从`TagStore`中查询或更新对应元素的污点标签。

原始数组对象本身没有任何改变，可以安全地传递给任何代码（包括原生方法），极大地提升了系统的稳健性。



### 方法调用间的标签传递

Galette为每个原始方法创建一个“影子方法”，该方法的参数列表是在原始参数基础上**追加一个`Frame`（标签帧）参数**。这个`Frame`对象包含了所有传入参数的污点标签，以及用于存放返回值的污点标签的空间。

所有方法调用点都被替换为对影子方法的调用，并传入构建好的`Frame`。

- 相比于镜像存储依赖“镜像调用栈”的隐式传递，这种**显式参数传递**确保了污点标签总能被准确地从调用者传递给正确的被调用者，即使JVM在中间执行了上行调用，也不会造成错位。

>镜像调用栈的挑战：
>
>- 由于镜像调用栈是独立于JVM栈的，它必须精确地模拟JVM的调用栈行为。但是，JVM在执行过程中可能会进行一些“上行调用”（up-calls）。这些上行调用可能会在原始程序指令之间插入，导致镜像调用栈的压栈和出栈操作与JVM栈不一致。
>- 例如，在仪器化代码中，可能在调用方法之前插入了将镜像帧压入镜像栈的代码，而在被调用方法开始处插入从镜像栈弹出帧的代码。但是，如果在这两者之间JVM执行了一个上行调用，那么这个上行调用也会被仪器化，并压入一个镜像帧，从而破坏了镜像栈的预期结构，导致污点标签传递错误。
>
>镜像存储方法的优点是非侵入性，不会改变原始程序的结构，但缺点是由于需要模拟整个执行环境，容易因模拟不精确而导致污点标签传播错误。
>
>**上行调用问题**：这是镜像调用栈的一个**致命弱点**。JVM在执行过程中，可能会在Java字节码指令之间插入对Java运行时本身的调用（例如，加载一个类、初始化一个类、抛出异常等）。这被称为“上行调用”。
>
>- **问题所在**：在仪器化代码刚刚压入一个镜像帧之后，但在目标方法实际执行之前，JVM进行了一个上行调用。这个上行调用也会被仪器化，并压入它自己的镜像帧。当目标方法最终开始执行时，它会错误地使用为上行调用准备的镜像帧，导致污点标签完全错乱。
>
>**性能开销**：维护一个完全独立的、与程序执行同步的镜像栈，并进行大量的映射查找和模拟操作，通常会带来比影子存储更高的性能开销。论文中的数据也显示，MirrorTaint的平均开销（5593倍）远高于Galette和Phosphor。



### 签名多态方法

**签名多态方法**是一类特殊的Java方法，它们**在源代码层面“看起来”可以接受任何参数类型和数量**，并且可以返回任何类型。

举个例子，他的声明方式：

```java
public final native @PolymorphicSignature Object invokeExact(Object... args) throws Throwable;
```



#### 它与普通方法的核心区别是什么？

| 特性                | 普通方法                                           | 签名多态方法                                                 |
| ------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| **类型检查时机**    | **编译时**。编译器根据方法声明的固定签名进行检查。 | **运行时**。在方法被**调用时**，JVM才根据调用点的具体类型描述符进行检查。 |
| **链接（Linking）** | 链接到一个固定的实现。                             | 链接到一个高度可变的、由方法句柄（MethodHandle）目标决定的实现。 |
| **方法签名**        | 固定的。                                           | **动态的**，取决于调用站点（call site）的写法。              |

在标准的Java类库中，主要有两个类包含签名多态方法：

- **`java.lang.invoke.MethodHandle`** (Java 7引入)
  - 例如：`invoke(Object... args)`, `invokeExact(Object... args)`, `invokeWithArguments(Object... args)`
- **`java.lang.invoke.VarHandle`** (Java 9引入)
  - 例如：`get(Object... args)`, `set(Object... args)`, `compareAndSet(Object... args)`

**无法修改签名**：传统的“影子存储”技术会通过修改方法签名（添加影子参数）来传递污点标签。但对于签名多态方法，这样做会**破坏其核心机制**。因为JVM会检查调用点的签名是否与 `MethodHandle` 的预期签名匹配，你添加了额外参数，匹配就会失败，调用无法进行。

**参数可能被JVM修改**：JVM在调用签名多态方法时，可能会**内部修改参数列表**（例如，添加或移除尾部的参数，或进行基本类型转换）。这使得污点跟踪系统难以在调用方和被调用方之间正确地对应参数和它们的污点标签。

签名多态方法的方法签名不能被修改，否则会导致类型匹配失败。

Galette的解决方案：采用一个线程局部的“帧存储”来间接传递标签。

1. 在调用签名多态方法前，插桩代码将参数的标签帧存入当前线程的帧存储中。
2. 然后，以原始方式（不修改签名）调用该多态方法。
3. 在**目标方法的一端**（任何一个方法都可能被多态调用，所以Galette将所有原始方法体都改造成了“包装器”），包装器会首先检查线程局部的帧存储。
4. 如果帧存储中有帧，并且通过一个“宽松的匹配算法”确认该帧属于本次调用，则使用该帧来初始化影子方法的参数。匹配算法会考虑JVM可能对参数做的固有修改（如添加或移除尾部参数）。
5. 最后，清除或恢复帧存储的状态。

什么意思呢？举个例子：

调用签名多态方法 `methodHandle.invokeExact(23, 45)`，其中 `23` 是被污染的，`45` 是干净的。

1、在调用具体的invokeExact方法之前，会先生成一个线程局部帧存储（只有当前线程能访问，避免帧错乱）

```java
ThreadLocalFrameStore.set(
    frame: [TAINTED, CLEAN], // 标签帧
    originalArgs: [23, 45]   // 帧数据
);
```

2、正常调用多态方法：调用原始的、未修改`invokeExact(23, 45)`，因为都是原始参数和方法，所以JVM运行正常

3、Galette加入了一个包装器方法，该方法会去第一步创建的线程局部帧存储看看有没有对应上的帧数据

```java
// 在包装器方法内部
Frame possibleFrame = ThreadLocalFrameStore.get(); // 去中转站查看
```

线程局部帧存储里可能有货，也可能没货。有货的话，这个货是不是就一定属于当前这个收货人呢？不一定！

这种情况当然发生于两个签名多态方法调用连续发生，并且它们共享同一个线程。但是什么代码会使它连续发生呢？

下面的mhFoo指向方法foo

```java
MethodHandle mhFoo = ...; // 指向 foo
// 第一个调用
mhFoo.invokeExact(...); 
```

在invokeExact发生了上行调用，调用到了类加载，刚好加载的那个类的static方法内有第二个签名多态方法

```java
MethodHandle mhBar = ...; // 指向 bar
static{
    mhBar.invokeExact(...);
}
```

就会发生第二个线程局部帧存储覆盖第一个的情况

当然这种情况实属罕见，作者能考虑到这种程度也真的很细心了

作者为了应付这种情况，特地加入了一个宽松匹配方法，因为作者认为，两次签名多态方法，总不能参数类型和数据一模一样吧

具体的宽松匹配算法：

* 场景A：**JVM加入了一个尾部参数**。假如线程局部帧存储里为如下数据：

```
[23, 45, SomeJVMStuff]
```

而包装器方法收到的`[23]`数据

宽松匹配算法会说："忽略那个JVM添加的额外东西，核心货物 `23` 对得上，这货是你的。"

* 场景B：**JVM转换了类型**。包装器收到一个 `int` 类型的 `23`，线程局部帧存储上记录的是一个 `byte` 类型的 `23`。宽松匹配算法会说："数值一样，只是包装不同，这货是你的。"

如果匹配成功，包装器则会取走标签帧 `[TAINTED, CLEAN]`，并清空线程局部帧存储；如果匹配失败，他会创建一个空的标签帧来处理

包装器现在有了标签帧（无论是有标签的还是空的），他终于可以去做真正的业务了。

他拿着收到的数据和标签帧，去调用**真正的后台处理部门（影子方法）**。

**这是一个典型的折中方案**：它引入了类似镜像存储的全局状态，因而在理论上仍然存在因上行调用导致标签错配的风险（尽管Galette通过匹配算法尽力缓解）。但这是在不破坏签名多态方法语义的前提下，能够跟踪信息流的唯一实用方法。



作者通过对各种污点传播过程中数据的存储方式进行了考量，完成了在JDK高版本下算得上是高兼容的污点数据结构。

## 源码分析

Gelette的前身为phosphor，这个项目经过了1200+commit，代码相当多，我看起来真的头痛

相比起来，Gelette 2025这个项目像个婴儿

### 预处理

premain起手，插桩类为GaletteTransformer

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125152331266.png)

插桩会调用GaletteTransformer#transform

具体来说就是去除无需插桩的java内部类，缓存已插桩的类，然后调用transformInternal进行具体的改造

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125152627011.png)

一句一个注释吗

### 影子字段和影子方法增强

跟进transformInternal，添加了Frame（标签帧）、影子字段和影子方法，具体的ASM代码就不看了

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125153049868.png)

跟进ShadowFieldAdder，可以看到加的影子字段前缀为`$$GALETTE`

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125154200966.png)

跟进ShadowMethodCreator，影子方法的处理会遍历该方法里用到的所有方法去生成影子方法。所以论文提到第一次污点attach会花费较长的时间，问题就出在这里，有时候一直遍历子方法可能会有bug

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125154308681.png)

原来看Dongtai IAST的时候我还没考虑到如果要增强JNI方法怎么办，事实上执行一遍用一个包装器去包装原JNI方法，然后外面套一个去掉native方法的壳就行了。这里也是这么处理的

先是去掉了方法的前置native和权限控制符，这些都会同时暂存起来。然后制作一个影子方法。那为什么没看到前面提到的帧存储呢？答案是放在了方法描述符里

这里完成影子方法的制作后，用TagPropagator完成一个方法中污点数据到另一个方法的传播（Propagator）

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125154906963.png)

在getShadowMethodDescriptor中，帧标签被一起放到方法描述符中存储

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125155118677.png)



### 操作树栈增强

ShadowLocals#visitFrame，拷贝原始局部变量到新的列表中，用TOP填补空位

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125160213781.png)

标签Tag，也就是污点传播的规则，全在TagPropagator里写死了

### source/sink

masks目录下是Node的规则，这里面既包含了source也包含了sink

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125161351659.png)

比如各种invoke，newInstance

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125161450967.png)

再比如各种序列化和反序列化

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125161529386.png)



本文的重点不在于污点传播规则的设定，而是一种新的字节码插桩方式。所以污点传播的指定也是只针对了他的benchmark

举个例子，他的ArrayAccessITCase测试三维数组元素的污点传播。测试排序后整型数组的污点标签是否正确保留。

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251125162838865.png)

感觉，不太实用，但是思路很好。纯科研算法吧。

没看完代码，看的我有点头痛，污点传播逻辑全用ASM写TagPropagator里，是个人都不想读了。不过根据bench来看，不像是用于漏洞测试，而是一个自定义的污点去测试系统功能。算是我最近读到能读懂，最贴合我的学习路线、思路最屌的论文了。
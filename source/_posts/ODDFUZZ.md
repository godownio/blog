---
title: "ODDFUZZ"
onlyTitle: true
date: 2025-12-23 21:52:48
categories:
- 杂七杂八
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8183.png
---



## ODDFUZZ

DOI:10.1109/SP46215.2023.10179377

ODDFUZZ: Discovering Java Deserialization Vulnerabilities via Structure-Aware Directed Greybox Fuzzing

感觉以后可以做这个，SP发表，虽然23年的文作者现在仓库仍为空，不过有点思路，值得记录

我来小巧的总结一下：

作者画了四五页讲CC链，这里不讲了。

实现方法就是：

1. 先定义和hook source、sink。至于用什么定义，我建议用OpenRasp-IAST写js脚本最方便，作者文中自己写的用的手写的Instrument。
2. 搜索source到sink的路径

创新点：**处理运行时多态（关键优化）**：

- 对于Java中的**虚方法调用**，传统方法（如类层次分析CHA）会盲目考虑所有可能的子类重写方法，极易导致“路径爆炸”。
- ODDFUZZ的优化在于：**仅当调用点（call site）本身被污染（即参数或`this`对象受Source影响）时，才会启用CHA去分析子类**。这个策略极大地减少了需要分析的无关路径。

与类似工具**GadgetInspector使用广度优先搜索（BFS）**不同，ODDFUZZ在程序调用图中，从标记的Source方法开始，**基于方法摘要进行深度优先搜索（DFS）**。

看懂这里先去看GadgetInspector

那么读到这里就有个疑问了，这里不是就知道Gadget了吗，知道gadget我还用这些花里胡哨的方法干嘛？接着往下看

算法中几个关键的部分：

* 关键一：前向种子突变策略

这是ODDFUZZ与传统随机变异的核心区别。传统Fuzzer随机翻转比特，极易产生语法无效的Java对象。而ODDFUZZ的变异是**在“属性树”的语义层面进行**的：

1. **属性级变异**：
   - 对于**基本类型**（如int）：调用 `random.nextInt()` 等方法生成随机值。
   - 对于**类类型**：从该属性所有可能的子类中，`random.choose()` 一个。
   - 对于**数组**：随机设置大小，并根据元素类型生成实例。
2. **前向引导**：
   - 这是算法的精髓。Fuzzer会分析当前种子卡在链中哪个Gadget方法里（利用执行反馈）。
   - 然后，它**有目的地突变**导致“卡住”的那个**嵌套子对象所对应的属性**。例如，如果执行卡在 `TransformingComparator.compare()`，算法就会翻转一个标识符，专门变异 `TransformingComparator` 类实例的属性（如 `transformer` 字段），而不是盲目变异整个对象。

* 混合反馈指导

>变异机制主要攻克了静态分析无法解决的三大难题：
>
>| 难题                      | 静态分析的局限                                               | 变异如何智能解决                                             |
>| ------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
>| **1. 满足复杂数据约束**   | 知道方法A调用方法B，但不知道调用B需要某个字段为特定值。      | **随机探索**：变异会不断尝试为字段赋予各种随机值（数字、字符串、对象），直到找到一个能让**程序流通过**的组合。 |
>| **2. 构建正确的对象类型** | 知道需要一个`Comparator`接口的实现，但不知道具体需要哪一个子类（如`TransformingComparator`）。 | **结构化选择**：变异时，会从一个候选类型池中，选择并实例化正确的子类对象来替换接口，从而满足运行时类型要求。 |
>| **3. 引导执行流向目标**   | 知道所有可能的岔路，但不知道哪条路最终能通向Sink。           | **定向引导**：结合“距离”反馈，变异会优先修改那些更有可能接近目标Sink的路径上的对象属性（前向种子突变）。 |

算法不是盲目变异，而是用两种反馈智能指导：

1. **种子距离**：衡量当前种子执行轨迹与目标Sink基本块之间的“距离”。距离越小的种子，被认为越有可能到达Sink，在下一轮中被分配更多**变异能量**（即更多次变异机会）。
2. **Gadget覆盖率**：衡量种子执行路径覆盖了多少个链上的Gadget方法。这有助于在初始阶段探索不同路径，避免陷入局部最优。

有个问题，这不是事先都知道当前种子执行轨迹与目标Sink块的距离了，意味着我不是都知道漏洞Sink和Gadget了，还找什么？

对，关键就是找什么

**ODDFUZZ不需要事先知道一条“完整且确定可利用”的链。它启动时所需的，是“通过轻量级静态分析推测出的、可能存在缺陷的候选链骨架”。**这个骨架是“可能”的路径，但其中充满了缺失的约束和不确定的分支。工具的核心任务，就是通过智能化的模糊测试，**动态地、自动化地为这个骨架“填充血肉”并验证其可行性。**

也就是说，事先用静态分析已经找出了Gadget（也就是前面说的BFS，我推测是修改了GadgetInspector的BFS为DFS

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251223213152922.png)

也就是修改GadgetInspector的BFS的出入栈为，，，等下？你告诉我GadgetInspector是什么遍历？？？

![](https://typora-202017030217.oss-cn-beijing.aliyuncs.com/typora/image-20251223212835060.png)

啥也没改吧你laodx，OK这里暂且不说

然后用ODDFUZZ来验证每个链子，也就是生成POC了。

不过能生成POC也是好事，减少了人工。

这里介绍一下文中提到的一个方法：JQF

### JQF

JQF 的核心是**为 Java 程序提供结构感知、覆盖引导的模糊测试**。

1. **不是“瞎变异”**：与 AFL 等直接变异字节的模糊测试器不同，JQF 强调**语义模糊测试**。它通过编写的 **`Generator`（生成器）** 来产生**语法有效**的输入（如合法的XML、JavaScript代码）。
2. **覆盖引导（Zest）**：JQF 的默认算法 Zest 会监控每次测试的**代码覆盖率**。它会优先保存和变异那些覆盖了新代码分支的输入，从而引导测试向程序更深的逻辑探索。
3. **属性测试框架**：只需像写单元测试一样，定义一个带有 `@Fuzz` 注解的方法，JQF 会自动为该方法参数生成值并反复运行它。

以 README 中的 `PatriciaTrieTest` 为例

```java
@RunWith(JQF.class) // 1. 指定使用JQF运行器
public class PatriciaTrieTest {
    @Fuzz // 2. 标记这是一个模糊测试方法，参数自动生成
    public void testMap2Trie(Map<String, Integer> map, String key) {
        // 3. 使用assumeTrue定义“有效输入”的假设
        assumeTrue(map.containsKey(key));
        // 4. 核心测试逻辑
        Trie trie = new PatriciaTrie(map);
        assertTrue(trie.containsKey(key)); // 断言：这就是我们要测试的属性
    }
}
```

- **`assumeTrue(...)`**：如果条件不满足，JQF 会**静默丢弃**当前输入，不视为失败。这用于引导工具生成更相关的数据。
- **`assertTrue(...)`**：如果断言失败，JQF 会**报告一个漏洞**，并保存导致失败的输入。

**运行模糊测试**：
在项目根目录（包含 `pom.xml`）下执行：

```bash
mvn jqf:fuzz -Dclass=PatriciaTrieTest -Dmethod=testMap2Trie
```

- JQF Maven 插件会启动模糊测试进程。
- 控制台会实时显示执行次数、覆盖率、发现的新路径等信息。
- 当断言失败（找到 bug）时，进程停止，并会在 `fuzz-results` 目录下生成导致失败的**测试用例文件**。

**重现崩溃**：找到 bug 后，可以使用以下命令确定性地重现它：

```bash
mvn jqf:repro -Dclass=PatriciaTrieTest -Dmethod=testMap2Trie -Dinput=fuzz-results/.../id_000000
```

#### 📝 构建您自己的测试驱动

对于更复杂的、需要自定义输入结构的场景（这正是ODDFUZZ所做的），需要：

**第一步：设置 Maven 依赖**
在 `pom.xml` 中添加 JQF 依赖和插件。

```xml
<dependency>
    <groupId>edu.berkeley.cs.jqf</groupId>
    <artifactId>jqf-fuzz</artifactId>
    <version>2.1</version> <!-- 使用最新版本 -->
    <scope>test</scope>
</dependency>
<!-- 同时需要引入junit -->
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
```

**第二步：编写自定义生成器 (Generator)**
当内置的随机生成器不满足需求时（例如需要生成特定的Java对象链），需要实现 `Generator<T>` 接口。

```java
public class MyComplexObjectGenerator implements Generator<MyComplexObject> {
    @Override
    public MyComplexObject generate(SourceOfRandomness random, GenerationStatus status) {
        // 使用random对象来构建的复杂对象
        MyComplexObject obj = new MyComplexObject();
        obj.setField1(random.nextChar());
        obj.setField2(random.nextBoolean());
        // ... 可能嵌套调用其他生成器
        return obj;
    }
}
```



**第三步：在测试中使用自定义生成器**
使用 `@From` 注解指定使用哪个生成器。

```java
@RunWith(JQF.class)
public class MyFuzzTest {
    @Fuzz
    public void testComplexLogic(@From(MyComplexObjectGenerator.class) MyComplexObject input) {
        // 对input进行假设和测试
        assumeTrue(input.isValid());
        myCodeUnderTest.process(input);
    }
}
```



**第四步：运行与分析**

- **运行**：同样使用 `mvn jqf:fuzz` 命令。
- **分析结果**：
  - **关注覆盖率增长**：覆盖率不再增长时，可能意味着测试已充分。
  - **检查 `fuzz-results/` 目录**：里面保存了独特的、触发新路径的测试用例。
  - **使用 `mvn jqf:repro`** 来调试任何发现的失败。

#### 🔧 其他运行方式与高级用法

1. **命令行直接运行 (Zest CLI)**：如果您已将测试打包成 JAR，可以直接用 `jqf-zest` 命令行工具运行，适合集成到 CI/CD。
2. **使用不同的引导算法 (Guidance)**：通过 `-Dengine` 参数可以切换不同的模糊测试引擎，例如 AFL 模式（适合二进制数据）。
3. **集成到 IDE**：您可以直接在 IDEA 或 Eclipse 中像运行普通 JUnit 测试一样运行 `@Fuzz` 测试，但这通常是用于调试，真正的模糊测试需要在命令行进行长时间运行。

GPT写的一个用JQF Fuzz CC链（我还没测试）：

需要实现 `Generator<Object>` 接口，其 `generate` 方法负责组装CC链

```java
public class CCChainGenerator implements Generator<Object> {
    private SourceOfRandomness random;
    
    @Override
    public Object generate(SourceOfRandomness random, GenerationStatus status) {
        this.random = random;
        // 1. 随机选择一个已知的CC链模板（例如：CommonsCollections1, 5, 6, 7）
        int chainType = random.nextInt(4); 
        
        // 2. 根据模板，调用相应方法构建对象树
        switch(chainType) {
            case 0:
                return buildTransformedMapChain(); // 类似CC1
            case 1:
                return buildLazyMapChain(); // 类似CC5
            // ... 其他链型
            default:
                return buildChainedTransformerChain();
        }
    }
    
    private Object buildTransformedMapChain() {
        // 核心：构建一个嵌套的 Transformer[] 和 Map 对象
        // 示例：模拟 CC1 的关键构造
        ChainedTransformer chain = new ChainedTransformer(new Transformer[] {
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod", 
                new Class[]{String.class, Class[].class}, 
                new Object[]{"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke", 
                new Class[]{Object.class, Object[].class}, 
                new Object[]{null, new Object[0]}),
            new InvokerTransformer("exec", 
                new Class[]{String.class}, 
                new Object[]{"calc.exe"}) // 无害化命令，如 echo test
        });
        
        HashMap<Object, Object> map = new HashMap<>();
        Map<Object, Object> lazyMap = LazyMap.decorate(map, chain);
        // ... 构造后续的TiedMapEntry和BadAttributeValueExpException等
        return lazyMap;
    }
}
```

编写JQF测试驱动 (`CCDeserializationTest`)

测试类负责接收生成器构造的对象，并尝试触发反序列化。

```java
@RunWith(JQF.class)
public class CCDeserializationTest {
    
    @Fuzz
    public void testDeserialize(@From(CCChainGenerator.class) Object candidateObject) {
        // 假设1: 对象非空且可序列化
        assumeTrue(candidateObject != null && candidateObject instanceof Serializable);
        
        try {
            // 1. 序列化
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(bos);
            oos.writeObject(candidateObject);
            oos.close();
            byte[] serializedData = bos.toByteArray();
            
            // 2. 反序列化 - 这是触发漏洞的关键！
            ByteArrayInputStream bis = new ByteArrayInputStream(serializedData);
            ObjectInputStream ois = new ObjectInputStream(bis);
            Object deserialized = ois.readObject(); // 如果漏洞存在，恶意代码可能在此处执行
            ois.close();
            
            // 3. (可选) 无害化断言，验证对象被成功还原，而非检测攻击成功
            // assertNotNull(deserialized);
            
        } catch (Exception e) {
            // 您可能期望某些异常（如ClassCastException），
            // 但一个成功的利用链会安静地执行命令，不抛异常。
            // 因此，这里更应监控进程的实际行为（如外部命令执行）。
        }
    }
}
```



```java
@Fuzz
public void testDeserializeWithMonitor(@From(CCChainGenerator.class) Object candidateObject) {
    // 清理监控痕迹
    new File("/tmp/jqf_success").delete();
    
    // ... 反序列化代码 ...
    
    // 反序列化后，检查命令是否被成功执行（文件是否存在）
    boolean exploitSuccess = new File("/tmp/jqf_success").exists();
    // 关键：告诉JQF，只有触发了命令执行的链才是“有趣”的输入
    assumeTrue(exploitSuccess);
    
    // 如果assumeTrue通过，JQF会保留此输入用于后续变异，从而集中资源寻找真正可利用的链
}
```

运行与调优

```bash
mvn jqf:fuzz -Dclass=CCDeserializationTest -Dmethod=testDeserializeWithMonitor
```

**调优参数**：

- `-Dtime=300`：设置模糊测试时间（秒）。
- `-Dtrials=100000`：设置最大测试次数。
- 观察**覆盖率增长**和**独特路径数**，它们是衡量探索效率的关键指标。

相当于JQF就是一个监测Gadget是否成功执行的东西，测试用例这一块

至于Fuzz的逻辑就写在switch(chainType)的函数里面，容易理解。



如果有科研后期我会写代码复现一下这个算法，如果有。。

我觉得可以创新的点：

1. 变异能量不是个好办法，因为一个字符型propagator变异很多次也是原来的东西，而他离sink很近，有多次变异机会也无济于事。离sink的距离其实跟变异机会（我理解的变异机会其实就等同于Gadget中单次对象生成成功率）没有任何联系，相反，如果一个propagator有很多字段，那么才应该增加变异机会
2. 从source到sink写payload从来都不是一个好的方案，我每次写payload都是从Runtime，TemplatesImpl开始写，如果一个Gadget的触发点是TemplatesImpl，那用这个工具一辈子也找不到POC，不如把搜集的Gadget看下有没有TemplatesImpl，然后从sink开始写。从sink往上找Gadget用这种变异去生成可用的对象，比从source可用性绝对大得多。
3. 既然都是逐步装填写poc了，大量的FUZZ用LLM肯定不行，但是最后POC的可用性和微小语法错误（变异生成的对象大概率有语法错误）可以用LLM检查top-k遍。

如果有人做了给我说一声别让我白费功夫


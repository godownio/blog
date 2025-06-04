---
title: "fastjson jackson getter setter和constructor的区分"
onlyTitle: true
date: 2024-10-25 15:23:23
categories:
- java
- 框架漏洞
tags:
- fastjson
- jackson
img: https://typora-202017030217.oss-cn-beijing.aliyuncs.com/%E5%9B%BE%E7%89%87%E7%B4%A0%E6%9D%90/1080P%20A%20%E6%94%B6%E8%97%8F%E9%87%8F%E6%9C%80%E5%A4%9A/1080PA%E5%A3%81%E7%BA%B8113.png
---



以下均指对json的反序列化过程

## setter

fastjson调用setter：遍历所有方法，找出所有满足setter要求的方法，再根据传入的json去反射调用

jackson调用setter：直接对传入的json中的字段拼接上set，然后去取setter

其实二者一样，但是fastjson不能调用私有和受保护setter，而jackson可以

## getter

fastjson调用getter：

1、没有setter方法，有getter方法。且getter满足：参数长度为0，返回类型是属于Collection 或其子类、Map 或其子类、AtomicBoolean、AtomicInteger、AtomicLong的一种

2、fastjson>=1.2.36 可用`$ref`调用getter

解析函数里有JSON.toJSON，还会调用getter（parseObject）

jackson调用getter：

1、没有该字段，没有setter，有getter方法，getter返回值是Map，Collection的子类。可以直接反序列化调用

2、jackson-databind&core&annotations>=2.13.3，通过POJONode调用getter

## 构造方法

fastjson调用构造方法：

* 如果存在无参构造方法，则将其作为构造方法
* 否则使用参数数量最多且排在最前面的构造方法。

jackson调用构造方法：

- 按照顺序，如果有参构造的参数能对应上，直接调用




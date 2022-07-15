---
title: "理解Gradle的所需的简单Groovy基础"
date: 2022-07-15T20:11:18+08:00
draft: true
toc: false
tags: []
---

最近对Gradle挺感兴趣的，为了更好的理解并使用Gradle，尤其是DSL背后的语法逻辑，简单的学习了下Groovy，记一个笔记。

Groovy的基础语法，数据类型，流程控制等和Java差不多，参照[Groovy的极简单教程](/java/groovy极简教程/)了解即可，此笔记主要介绍这3个内容：

1. DSL
2. DGM
3. Closure

## 1. DSL

Groovy之所以适合于构建DSL，就是因为它**可以在调用方法时省略括号**。这个特性再结合上**链式调用**，就使得代码看起来像配置。一段`a b c d`这样的Groovy代码，其实等价于这样的普通写法:`a(b).c(d)`，`a`和`c`是方法，而`b`和`d`分别是传给`a`和`c`的参数，显然，`a(b)`的返回值有一个`c`方法。第一此见到这样的写法时心中立马就有个疑问，如果`a`不止有1个参数，比如`a`有2个参数，那么`a b c d`是等价于`a(b, c).d`吗？其实不是，有多个参数的情况下，参数之间需要用逗号分割。比如`a b, c d e, f`等价于`a(b, c).d(e, f)`，一般这样写起来就不太直观了，个人觉得就直接按普通的写法写好了。

使得Groovy适合于构建DSL当然还有其他的因素，比如操作符重载，闭包，委托等，就不描述了。

## 2. DGM

DGM即`DefaultGroovyMethods`，每个Groovy对象都‘自动’的拥有很多方法（几十上百个），就像Java中所有的对象都是`java.lang.Object`的子类，就自动拥有`toString`，`hashCode`等方法，不同的是通过DGM拥有的方法背后不是继承来的，而应该是代理来的，所以通过`getClass()`拿到`Class`对象后也看不到这些方法。

具体有那些DGM方法，以及怎么使用参见文档：[DefaultGroovyMethods](https://docs.groovy-lang.org/latest/html/api/org/codehaus/groovy/runtime/DefaultGroovyMethods.html)

了解这个后就不会因为在Gradle代码中看到一个代码看起来像调用了一个不存在的方法而疑惑了。

## 3. Closure

```
A closure in Groovy is an open, anonymous, block of code that can take arguments, 
return a value and be assigned to a variable. A closure may reference variables 
declared in its surrounding scope. In opposition to the formal definition of a closure,
Closure in the Groovy language can also contain free variables which are defined 
outside of its surrounding scope. While breaking the formal concept of a closure, it offers a 
variety of advantages which are described in this chapter.
```

闭包就是一段可以接收参数的代码块，并且可以返回值，闭包可以当成函数调用的参数，这样函数调用就看起来像DSL。

一个闭包对象的定义语法如下：

```
{ [closureParameters -> ] statements }
```

`[closureParameters -> ]`是可选的逗号分割的参数列表，当没有用`->`显式的定义参数时，闭包有一个名为`it`的**隐藏参数**。

`{ println it }`等价于`{ it -> println it }`

与闭包相关的还有一个委托（delegate）的概念, 为了理解`delegate`，先要搞清楚定义一个闭包是涉及到的以下3个概念：

- **this** 

    `this` corresponds to the enclosing class where the closure is defined

- **owner** 

    `owner` corresponds to the enclosing object where the closure is defined, which may be either a class or a closure

- **delegate** 

    `delegate` corresponds to a third party object where methods calls or properties are resolved whenever the receiver of the message is not defined

这个参考官方文档的例子代码来理解。

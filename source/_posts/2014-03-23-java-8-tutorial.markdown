---
layout: post
title: "Java8指南[Java 8 Tutorial]"
date: 2014-03-23 10:48
comments: true
keywords: java,java 8,java8
categories: 
- Java
- 技术
---
{% img center /images/2014/03/java8.png %}

本文翻译自[Java 8 Tutorial](http://winterbe.com/posts/2014/03/16/java-8-tutorial/)    
>["Java还没死-大家正在纷纷议论"](https://twitter.com/mreinhold/status/429603588525281280)  

首先欢迎大家来看我对Java8的介绍。本文将引导你一步步的了解所有新的语言特性。利用简短的代码片段，你将学习如何去使用`缺省接口方法（default interface methods）`,`lambda 表达式`,`方法引用（method references）`和`可重复注解（repeatable annotations）`.当你看到本文结束的时候，你将非常熟悉像`Streams`,`函数式接口`,`map 扩展`和`新的Date API`API的变化。

这里没有整篇的文字-仅仅是一系列注释的代码片段。让我们开启本文的愉快之旅吧。

#接口缺省方法（Default Methods for Interfaces）

Java8容许通过`default`关键字给接口增加非抽象的方法实现,这个特性也可以叫做`扩展方法(Extension Methods)`.这里是我们第一个例子：   
```java

interface Formula {
    double calculate(int a);

    default double sqrt(int a) {
        return Math.sqrt(a);
    }
}
```
<!-- more -->

除了calculate抽象方法以外，Formula接口还定义了一个缺省方法`sqrt`，这样以来，实现类就只需要实现抽象方法即可，缺省方法`sqrt`可以不用重新实现拿来即用。

```java
Formula formula = new Formula() {
    @Override
    public double calculate(int a) {
        return sqrt(a * 100);
    }
};

formula.calculate(100);     // 100.0
formula.sqrt(16);           // 4.0
```
`formula`通过匿名对象实现，上面的代码非常冗长：用了6行的代码实现了如此简单的`sqrt(a*100)`的计算。正如我们即将在接下来的部分所了解的，在Java8中，我们可以通过更加漂亮的方式来实现单一方法对象。  

#Lambda表达式(Lambda expressions)
让我们用一个之前版本Java中排序一个字符串列表的简单例子开始：
```java
List<String> names = Arrays.asList("peter", "anna", "mike", "xenia");

Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return b.compareTo(a);
    }
});
```
静态工具方法`Collections.sort`接受一个列表和比较器对象去排序一个指定列表的元素。你通常创建一个匿名的比较器并将其传递sort方法。

你不需要总是创建匿名对象，Java8中你可以用一种更加简洁的语法，lambda表达式：
```java
Collections.sort(names, (String a, String b) -> {
    return b.compareTo(a);
});
```
正如你看到上面的代码更加简短并且易读，但是它可以变得更加简短：
```java
Collections.sort(names, (String a, String b) -> b.compareTo(a));
```
对于只有一行代码的方法，你可以不用{}和return关键字，但是它还可以变得更简短：
```java
Collections.sort(names, (a, b) -> b.compareTo(a));
```
Java编译器清楚参数的类型，所以你也可以省略他们。接下来，让我们更加深入的来了解一下lambda表达式的使用。

.....未完待续......

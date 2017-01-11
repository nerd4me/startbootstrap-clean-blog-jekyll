---
layout:     post
title:      "Guava指南"
subtitle:   "Guava Tutorial「译」"
date:       2017-01-11 20:19:00 +0800
author:     "Nerd4me"
catalog:    true
tags:
    - guava
---

## 介绍

Guava是Google开发的一款基于Java的开源工具库，它被广泛用于Google的其他项目，它有助于提供最佳编码实践及减少编码错误。它提供了大量的工具函数，如：集合、缓存、原生类型支持、并发、通用注解、字符串处理、I/O、校验等。

## 使用Guava的好处

- ***标准化*** Guava项目由Google统一管理，并托管在[GitHub](https://github.com/google/guava)
- ***高效*** 对Java标准库可靠、快速、高效的扩展
- ***高度优化*** 高度优化实现
- ***函数式编程*** Java函数式编程实现
- ***实用工具*** 大量应用程序开发常用工具类实现
- ***验证*** 标准化故障安全验证机制
- ***最佳实践*** 强调最佳实践

考虑如下代码片段。
```java
public class GuavaTester {
    public static void main(String[] args) {
        GuavaTester guavaTester = new GuavaTester();

        Integer a = null;
        Integer b = 10;

        System.out.println(guavaTester.sum(a, b));
    }

    public Integer sum(Integer a, Integer b) {
        return a + b;
    }
}

/*
 * 程序运行结果：
 * Exception in thread "main" java.lang.NullPointerException
 *  at GuavaTester.sum(GuavaTester.java:13)
 *  at GuavaTester.main(GuavaTester.java:9)
 */
```

上述代码存在的问题：
- sum()函数不对任何传入的有可能为空的参数做空值校验
- 调用方法也未对传入函数的变量做空值校验
- 程序运行时抛出`NullPointerException`

为了避免上述问题，对每个可能存在这些问题的地方都应该进行空值检查。

Guava提供的工具类`Optional`为解决上述问题提供了一种标准化的方法，让我们看看它是如何做到的。

```java
public class GuavaTester {
    public static void main(String[] args) {
        GuavaTester guavaTester = new GuavaTester();
        Integer invalidInput = null;
        Optional<Integer> a = Optional.of(invalidInput);
        Optional<Integer> b = Optional.of(10);

        System.out.println(guavaTester.sum(a, b));
    }

    public Integer sum(Optional<Integer> a, Optional<Integer> b) {
        return a.get() + b.get();
    }
}

/*
 * 程序运行结果：
 * Exception in thread "main" java.lang.NullPointerException
 *   at com.google.common.base.Preconditions.checkNotNull(Preconditions.java:210)
 *   at com.google.common.base.Optional.of(Optional.java:85)
 *   at GuavaTester.main(GuavaTester.java:8)
 */
```

上述程序中重要的概念如下：

- Optional Guava提供的工具类，帮助我们在代码中正确的使用`null`。
- Optional.of 返回将要做为函数参数的Optional类实例，它将检查传入的参数，不能为`null`。
- Optional.get 获取保存在Optional实例中的参数值。

通过使用`Optional`类，你可以检查调用方法是否传入正确的参数值。

## Optional

#### 类定义
如下是`com.google.common.base.Optional<T>`的类定义：
```java
@GwtCompatible(serializable=true)
public abstract class Optional<T>
   extends Object
      implements Serializable
```

#### 类方法

|Sr.No| Method & Description                                            |
|:----|:----------------------------------------------------------------|
|1    | `static <T> Optional<T>  absent()`<br />*返回一个不包含值的Optional对象引用.*                                      |
|2    | `abstract Set<T> asSet()`<br />*返回一个只包含当前对象代表元素的不可变单例`Set`，如果没有，则返回空`Set`..*                                      |
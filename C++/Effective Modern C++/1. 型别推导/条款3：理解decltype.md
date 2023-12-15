---
tags:
  - cplusplus decltype
---

# 1. 初始decltype

## 1.1 概述

***decltype***是declare type(声明类型)的缩写，是C++11新增的一个关键字，它与***auto***的功能都一样，都用来在编译期间进行自动型别/类型推导。那么既然已经有了***auto*** 关键字，为什么还需要 ***decltype*** 关键字呢？这是因为 ***auto*** 并不适用于所有的自动类型推导场景，在某些特殊情况下 ***auto*** 用起来非常不方便，甚至压根无法使用，所以 ***decltype*** 关键字也被引入到 C++11 中。

## 1.2 基本概念

***decltype*** 的基本用法如下：

``` C++ 伪代码

decltype(expression) // 其中expression为任意表达式，该语句的含义是返回expression的类型

int a = 0;
decltype(a) b = 1; // b会被推导为int型
decltype(1.234f) c = 1.0f; // c会被推导为float型
decltype(c + 100) d = 2.345f; // d会被推导为float型

```

有上述几个示例，可以看到的是 ***decltype*** 的作用是推导出表达式的类型。

## 1.3 型别/类型推导规则

既然 ***decltype*** 的作用也是进行型别/类型推导，这也就意味着它也有自己的型别/类型推导规则，具体来说型别推导规则如下：

- 如果表达式 ***expression*** 参数是不带括号()的简单标识符，或者是某个类的成员变量，那么 ***decltype(expression)*** 就是表达式***expression*** 的类型。如果并不存在 ***expression*** 这样的实体对象，或者 ***expression*** 描述的是一组经过重载(overloaded)的函数名(此时返回值类型可能有多个)，则会触发编译报错。
- 如果表达式 ***expression*** 是一个对函数或者重载运算符函数的调用，那么 ***decltype(expression)*** 表示的是函数的返回值类型，这里需要注意因为括号()比较特殊，因此此处所说的运算符重载不包括对括号()的重载。
- 如果表达式 ***expression*** 是一个右值那么 ***decltype(expression)*** 表示的是 ***expression*** 的类型。如果表达式 ***expression*** 是一个左值，***decltype(expression)*** 表示的是 ***expression*** 的左值引用类型。
- 如果表达式 ***expression*** 是一个被括号()修饰的表达式
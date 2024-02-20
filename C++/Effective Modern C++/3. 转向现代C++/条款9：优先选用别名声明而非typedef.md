---
tags:
  - cplusplus modern-cplusplus typedef alias-declaration
---

# 1. 概述

当我们使用C++编写代码时，常常遇到复杂结构类型的定义和使用( eg: `std::unique_ptr<std::unordered_map<std::string, std::string>>`) ，为了避免我们重复书写这么复杂的类型定义/声明，C++为我们提供了语法糖可以使用另外的自定义名来代替此复杂类型。在C++98中有 `typedef`，在C++11中则提供了一个别名声明（_alias declaration_）。

``` C++

typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMapSS;  // C++98语法 使用UPtrMapSS代替此复杂结构的声明

using UPtrMapSS = std::unique_ptr<std::unordered_map<std::string, std::string>>; // C++11语法 将UPtrMapSS设置为复杂结构的别名

```

`typedef`和别名除了可以用于上述复杂结构的声明外还可以用于函数指针的声明：

``` C++

//FP是一个指向函数的指针的同义词，它指向的函数带有
//int和const std::string&形参，不返回任何东西
typedef void (*FP)(int, const std::string&);    //typedef

//含义同上
using FP = void (*)(int, const std::string&);   //别名声明

```

那么，既然`typedef`和别名都可以实现同样的效果，那他们又有何不同呢？或者说既然C++98已经有了`typedef`为什么在C++11中还要引入别名呢？
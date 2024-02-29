---
tags:
  - cplusplus modern-cplusplus private
---

# 1. 概述

在使用C++编程时，有时我们希望使用我们代码的人不要调用某些函数，或者说禁止其他用户调用和访问某些函数。可能你会觉得既然不想被其他人调用和使用只要我们不写(不去声明和定义)这个函数就可以了。但是C++会在需要的时候为某些特殊的函数自动生成成员函数，例如：赋值构造函数，赋值函数等([[条款17：理解特种成员函数的生成机制|条款17]]中会对此类函数进行详细讨论)。此时你可能又会想，既然这样，我们将这些函数声明为私有的(`private`)就可以避免其他用户的访问和使用。

在C++98中确实可以通过将函数声明为私有的(`private`)，并且不提供任何该函数的定义/实现来避免其他用户的调用和使用。将函数声明为私有的(`private`)，是为了阻止用户去调用它，而不提供任何函数定义/实现，则是为了当有代码调用它时编译器可以提供显示的报错提示(`not defined`)。例如，在标准库中，输入/输出流类(`istream/ostream`)的基类是`basic_ios`，对于这种读写数据流的类进行复制是不可取的，因为这种操作究竟要落实成什么样子的行为是不确定的。因此我们需要在基类`basic_ios`中限制拷贝构造和赋值操作：

``` C++

template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …

private:
    basic_ios(const basic_ios& );           // not defined
    basic_ios& operator=(const basic_ios&); // not defined
};

```

而在C++11中则有了更好的方式来实现以上效果：使用 `= delete`将赋值构造和复制运算符编辑为**删除函数**(_deleted function_)：

``` C++

template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
    …

    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
    …
};

```


# 2. 删除函数的优势及好处

## 2.1 更加全面的禁止调用

表面上看，使用**删除函数**(_deleted function_)与C++98中的实现方式只是写法上的不同，然而，实际上还有更多的细节和值得思考之处。

C++98中的写法并不能避免类对象内部以及友元类对象对此函数的调用，并且在这种情况下的调用只要到了链接阶段才会被编译器诊断出来(因为只有私有函数的声明而并没有具体实现)。但使用 `= delete`将函数编辑为删除函数的方式则会使得该函数无法通过任何方法调用使用。

通常，_deleted_ 函数被声明为`public`而不是`private`。这也是有原因的: 当用户代码试图调用成员函数，C++会先检查函数的可访问性(`public or private`), 然后才检查 _deleted_ 状态。这也就意味着当用户代码调用一个私有的_deleted_函数，一些编译器只会给出该函数是`private`的错误而不会给出诸如该函数被_deleted_修饰的错误。所以值得牢记，如果要将老代码的“私有且未定义”函数替换为_deleted_函数时请一并修改它的访问性为`public`，这样可以让编译器产生更好的错误信息。

## 2.2 更加广阔的适用范围


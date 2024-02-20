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

那么，既然`typedef`和别名都可以实现同样的效果，那他们又有何不同呢？或者说既然C++98已经有了`typedef`为什么在C++11中还要引入别名呢？这是因为C++11中的别名比`typedef`适用范围更广也更有优势，因此我们推荐优先选用别名而非`typedef`。

对于模板的支持，别名比`typedef`更有优势：别名声明可以被模板化（这种情况下称为别名模板_alias templates_）但是`typedef`不能。这使得C++11程序员可以很直接的表达一些C++98中只能把`typedef`嵌套进模板化的`struct`才能表达的东西。考虑一个链表的别名，链表使用自定义的内存分配器，`MyAlloc`。使用别名模板就比使用`typedef`简单方便很多：

``` C++

template<typename T>                            //MyAllocList<T>是std::list<T, MyAlloc<T>>的同义词 
//使用typedef定义模板必须包一层，无法直接用于模板
struct MyAllocList 
{                            
    typedef std::list<T, MyAlloc<T>> type;     
};

MyAllocList<Widget>::type lw;          //用户代码


template<typename T>                            //MyAllocList<T>是std::list<T, MyAlloc<T>>的同义词
using MyAllocList = std::list<T, MyAlloc<T>>;   

MyAllocList<Widget> lw;                         //用户代码

```

更糟糕的是，如果你想使用在一个模板内使用`typedef`声明一个链表对象，而这个对象又使用了模板形参，你就不得不在`typedef`前面加上`typename`：

``` C++

template<typename T>
class Widget {                              //Widget<T>含有一个
private:                                    //MyAllocLIst<T>对象
    typename MyAllocList<T>::type list;     //作为数据成员
    …
}; 

```

这里`MyAllocList<T>::type`使用了一个类型，这个类型依赖于模板参数`T`。因此`MyAllocList<T>::type`是一个依赖类型（_dependent type_），在C++很多讨人喜欢的规则中的一个提到必须要在依赖类型名前加上`typename`。

如果使用别名声明定义一个`MyAllocList`，就不需要使用`typename`（同时省略麻烦的“`::type`”后缀）：

``` C++

template<typename T> 
using MyAllocList = std::list<T, MyAlloc<T>>;   //同之前一样

template<typename T> 
class Widget {
private:
    MyAllocList<T> list;             //没有“typename”
    …                                //没有“::type”
};

```

产生这样结果的原因时：对我们来说，`MyAllocList<T>`（使用了模板别名声明的版本）可能看起来和`MyAllocList<T>::type`（使用`typedef`的版本）一样都应该依赖模板参数`T`，但是我们不是编译器。当编译器处理`Widget`模板时遇到`MyAllocList<T>`（使用模板别名声明的版本），它知道`MyAllocList<T>`是一个类型名，因为`MyAllocList`是一个别名模板：它**一定**是一个类型名。因此`MyAllocList<T>`就是一个**非依赖类型**（_non-dependent type_），就不需要也不允许使用`typename`修饰符。但当编译器在`Widget`的模板中看到`MyAllocList<T>::type`（使用`typedef`的版本），它不能确定那是一个类型的名称。因为可能存在一个`MyAllocList`的它们没见到的特化版本，那个版本的`MyAllocList<T>::type`指代了一种不是类型的东西。那听起来很不可思议，但不要责备编译器穷尽考虑所有可能。因为人确实能写出这样的代码。

``` C++

class Wine { … };

template<>                          //当T是Wine
class MyAllocList<Wine> {           //特化MyAllocList
private:  
    enum class WineType             //参见Item10了解  
    { White, Red, Rose };           //"enum class"

    WineType type;                  //在这个类中，type是
    …                               //一个数据成员！
};

```

假设你写出了上述代码`MyAllocList<Wine>::type`不是一个类型。如果`Widget`使用`Wine`实例化，在`Widget`模板中的`MyAllocList<Wine>::type`将会是一个数据成员，不是一个类型。在`Widget`模板内，`MyAllocList<T>::type`是否表示一个类型取决于`T`是什么，这就是为什么编译器会坚持要求你在前面加上`typename`。

这一弊端，在`std`标准库中也有所体现：如果我们写过模板元编程(_template metaprogramming_，**TMP**)，我们一定熟悉C++11中的_type traits_。_type traits_（类型特性）中为我们提供了一系列工具去实现类型转换(`const`到非`const `引用&与非引用之前的转换等)。

``` C++

std::remove_const<T>::type          //从const T中产出T
std::remove_reference<T>::type      //从T&和T&&中产出T
std::add_lvalue_reference<T>::type  //从T中产出T&

```

注意类型转换尾部的`::type`。如果我们在一个模板内部将他们施加到类型形参上（实际代码中我么也总是这么用），我们也需要在它们前面加上`typename`。至于为什么要这么做是因为这些C++11的_type traits_是通过在`struct`内嵌套`typedef`来实现的。

关于为什么这么实现是有历史原因的，但是我们跳过它（我认为太无聊了），因为标准委员会没有及时认识到别名声明是更好的选择，所以直到C++14它们才提供了使用别名声明的版本。这些别名声明有一个通用形式：对于C++11的类型转换`std::`transformation`<T>::type`在C++14中变成了`std::`transformation`_t`。举个例子或许更容易理解：

``` C++

std::remove_const<T>::type          //C++11: const T → T 
std::remove_const_t<T>              //C++14 等价形式

std::remove_reference<T>::type      //C++11: T&/T&& → T 
std::remove_reference_t<T>          //C++14 等价形式

std::add_lvalue_reference<T>::type  //C++11: T → T& 
std::add_lvalue_reference_t<T>      //C++14 等价形式

```

C++11的的形式在C++14中也有效，但是我不能理解为什么你要去用它们。就算你没办法使用C++14，使用别名模板也是小儿科。只需要C++11的语言特性，甚至每个小孩都能仿写，对吧？如果你有一份C++14标准，就更简单了，只需要复制粘贴：

``` C++

template <class T> 
using remove_const_t = typename remove_const<T>::type;

template <class T> 
using remove_reference_t = typename remove_reference<T>::type;

template <class T> 
using add_lvalue_reference_t =
  typename add_lvalue_reference<T>::type; 

```

这样问题就会变得更加简单，所以尽量选用别名而不是`typedef`，因为对于模板而已别名更加简单方便且无需为思考是否该使用`typename`而发愁。

# 2. 要点速记

- `typedef`不支持模板化，但是别名声明支持。
- 别名模板避免了使用“`::type`”后缀，而且在模板中使用`typedef`还需要在前面加上`typename`
- C++14提供了C++11所有_type traits_转换的别名声明版本
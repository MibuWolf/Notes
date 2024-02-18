---
tags:
  - cplusplus modern-cplusplus nullptr
---

# 1. 概述

当我们要表示空指针时，应优先选用 `nullptr` 而非 `0` 或者 `NULL`。这是因为本质上只有 `nullptr` 才是指针类型，字面值`0`是一个`int`而不是指针，C++只有在发现当前上下文只能使用指针时，它才会很不情愿的把`0`解释为指针，但是那是最后的退路。一般来说C++的解析策略是把`0`看做`int`而不是指针。实时上 `NULL` 也是这样的。但在`NULL`的实现细节有些不确定因素，因为实现被允许给`NULL`一个除了`int`之外的整型类型（比如`long`）

在C++98中，对指针类型和整型进行重载意味着可能导致奇怪的事情。如果给下面的重载函数传递`0`或`NULL`，它们绝不会调用指针版本的重载函数：

``` C++

void f(int);        //三个f的重载函数
void f(bool);
void f(void*);

f(0);               //调用f(int)而不是f(void*)

f(NULL);            //可能不会被编译，一般来说调用f(int)，
                    //绝对不会调用f(void*)

```

^a70f19

而`f(NULL)`的不确定行为是由`NULL`的实现不同造成的。如果`NULL`被定义为`0L`（指的是`0`为`long`类型），这个调用就具有二义性，因为从`long`到`int`的转换或从`long`到`bool`的转换或`0L`到`void*`的转换都同样好。
有趣的是源代码**表现出**的意思（“我使用空指针`NULL`调用`f`”）和**实际表达出**的意思（“我是用整型数据而不是空指针调用`f`”）是相矛盾的。这种违反直觉的行为导致C++98程序员都将避开同时重载指针和整型作为编程准则（注：请务必注意结合上下文使用这条规则）。在C++11中这个编程准则也有效，因为尽管本条款建议使用`nullptr`，可能很多程序员还是会继续使用`0`或`NULL`，哪怕`nullptr`是更好的选择。

`nullptr`的优点是它不是整型。严格说它也不是一个指针类型，但是你可以把它认为是**所有**类型的指针。`nullptr`的真正类型是`std::nullptr_t`，在一个完美的循环定义以后，`std::nullptr_t`又被定义为`nullptr`。`std::nullptr_t`可以隐式转换为指向任何内置类型的指针，这也是为什么`nullptr`表现得像所有类型的指针。

使用 `nullptr` 可以解决[[条款8：优先选用nullptr而非0或NULL#^a70f19|示例代码]]中不同版本重载函数的问题，这是因为 `nullptr`能够隐式转化成 `void*` 类型但又不能被视作任何整型：

``` C++

f(nullptr);         //调用重载函数f的f(void*)版本

```

使用 `nullptr` 除了可以解决上述函数重载问题，它还可以更加直观的表明代码含义，尤其是当与 `auto` 一起使用时：

``` C++

auto result = findRecord( /* arguments */ );
if (result == 0) {   // 用0表示空指针，不能很好的阅读理解出result
                     // 变量的真实类型
    …
} 

if (result == nullptr) {  // 使用nullptr就可以很清晰的知道result是
                          // 一个对象的指针
    …
}

```

此外，当我们使用模板时 `nullptr` 就会变得更有用了，假设我们有一组函数只能被合适的已锁互斥量调用。每个函数都有一个不同类型的指针：

``` C++

int    f1(std::shared_ptr<Widget> spw);     //只能被合适的
double f2(std::unique_ptr<Widget> upw);     //已锁互斥量
bool   f3(Widget* pw);                      //调用

```

当我们需要在多线程中互斥的执行这些函数时，就需要类似如下的方式进行调用：

``` C++

std::mutex f1m, f2m, f3m;       //用于f1，f2，f3函数的互斥量

using MuxGuard =                //C++11的typedef，参见Item9
    std::lock_guard<std::mutex>;
…

{  
    MuxGuard g(f1m);            //为f1m上锁
    auto result = f1(0);        //向f1传递0作为空指针
}                               //解锁 
…
{  
    MuxGuard g(f2m);            //为f2m上锁
    auto result = f2(NULL);     //向f2传递NULL作为空指针
}                               //解锁 
…
{
    MuxGuard g(f3m);            //为f3m上锁
    auto result = f3(nullptr);  //向f3传递nullptr作为空指针
}                               //解锁 

```

用此方式单独的一个个调用每个函数固然没有问题，但如果我们想更进一步的利用模板减少重复代码时，可能我们需要封装一个模板来实现加锁，执行函数，解锁的全过程：

``` C++

template<typename FuncType,
         typename MuxType,
         typename PtrType>
auto lockAndCall(FuncType func,                 
                 MuxType& mutex,                 
                 PtrType ptr) -> decltype(func(ptr))
{
    MuxGuard g(mutex);  
    return func(ptr); 
}

```

如果你对函数返回类型（`auto ... -> decltype(func(ptr))`）感到困惑不解，回顾下[[条款3：理解decltype#^bce4ef|条款3中的尾置返回类型语法]]。在C++14中代码的返回类型还可以被简化为`decltype(auto)`：

``` C++

template<typename FuncType,
         typename MuxType,
         typename PtrType>
decltype(auto) lockAndCall(FuncType func,       //C++14
                           MuxType& mutex,
                           PtrType ptr)
{ 
    MuxGuard g(mutex);  
    return func(ptr); 
}

```

当调用此模板函数来实现 加锁 函数调用 解锁时，`nullptr`的优势就更为明显了：

``` C++

auto result1 = lockAndCall(f1, f1m, 0);         //错误！
...
auto result2 = lockAndCall(f2, f2m, NULL);      //错误！
...
auto result3 = lockAndCall(f3, f3m, nullptr);   //没问题

```

当调用 `lockAndCall` 模板函数时，参数 `0` 或 `NULL` 被传递给`lockAndCall`时会尝试进行实参类型推导，不幸的是，这意味着`lockAndCall`中`func`会被`int`类型的实参调用，这与`f1`期待的`std::shared_ptr<Widget>`形参不符。传递`0`给`lockAndCall`本来想表示空指针，但是实际上得到的一个普通的`int`。把`int`类型看做`std::shared_ptr<Widget>`类型给`f1`自然是一个类型错误。`NULL`也是一样的问题，然而，使用`nullptr`是调用没什么问题。当`nullptr`传给`lockAndCall`时，`ptr`被推导为`std::nullptr_t`。当`ptr`被传递给`f3`的时候，隐式转换使`std::nullptr_t`转换为`Widget*`，因为`std::nullptr_t`可以隐式转换为任何指针类型。

模板类型推导将`0`和`NULL`推导为一个错误的类型（即它们的实际类型，而不是作为空指针的隐含意义），这就导致在当你想要一个空指针时，它们的替代品`nullptr`很吸引人。使用`nullptr`，模板不会有什么特殊的转换。另外，使用`nullptr`不会让你受到同重载决议特殊对待`0`和`NULL`一样的待遇。当你想用一个空指针，使用`nullptr`，不用`0`或者`NULL`。

# 2. 要点速记

- 优先考虑`nullptr`而非`0`和`NULL`
- 避免重载指针和整型
---
tags:
  - cplusplus auto
---

# 1. 概述

当我们阅读完[[条款1： 理解模板型别推导|模板型别推导]]规则，那么我们也已经了解了auto型别推导的基础规则了。可能你会感到疑惑和模板型别(类型)推导相关的是：函数，形参，实参这些要素，而auto跟它们毫不相干。确实如此，auto没有函数也没有形参，实参，但这并不妨碍他们使用同样一套的推导规则，不同点仅仅在于[[条款2：理解auto型别推导#2. 对于大括号初始化表达式的型别推导|对于大括号初始化表达式的型别推导]]。
为了方便我们理解auto型别推导规则，我们可以将auto的推导过程想象成对一个概念模板的型别推导。所谓的概念模板，并不是说编辑器诊断生成了这样一个模板和语句，仅仅是为了帮助我们理解而虚构出来的概念上的模板语句。在[[条款1： 理解模板型别推导|模板型别推导]]中我们知道对模板型别的推导本质上 是：编译器利用[[条款1： 理解模板型别推导#^4c803b|expr来推导T和ParamType的型别(类型)]]。因此当我们分析auto的型别推导时，我们就可以用概念模板的方式帮助我们理解其推导过程，如下伪代码所示：

``` C++ 伪代码

auto x = 27;
const auto cx = x;
const auto & rx=cx;

```

在分析上述代码中auto的型别(类型)推导时，我们使用概念模板把auto想象成一个模板函数以方便我们理解，因此可以将上述三行代码想象成为三个模板函数，如下所示：

``` C++ 伪代码

template<typename T>            //概念化的模板用来推导x的类型
void func_for_x(T param);

func_for_x(27);                 //auto x = 27的概念化调用：
                                //param的推导类型是x的类型

template<typename T>            //概念化的模板用来推导cx的类型
void func_for_cx(const T param);

func_for_cx(x);                 //const auto cx =x  的概念化调用：
                                //param的推导类型是cx的类型

template<typename T>            //概念化的模板用来推导rx的类型
void func_for_rx(const T & param);

func_for_rx(x);                 //const auto & rx=cx的概念化调用：
                                //param的推导类型是rx的类型

```

经过这样在概念上将其转化为模板函数(为了方便理解想象出来的，实际上编译器并不是这样分析的)可以很简单的推导出auto的型别类型。
在[[条款1： 理解模板型别推导|模板的型别推导]]中，我们分了三种情况来进行讨论，这三种情况同样的适用于auto的型别推导，如下示例代码所示：

``` C++ 伪代码

auto x = 27;              // 情况三（x既不是指针也不是引用）
const auto cx = x;              //情景三（cx也一样）
const auto & rx=cx;             //情景一（rx是非通用引用）

auto&& uref1 = x;               //情景二 x是int左值，
                                //所以uref1类型为int&
auto&& uref2 = cx;              //情景二 cx是const int左值，
                                //所以uref2类型为const int&
auto&& uref3 = 27;              //情景二 27是int右值，
                                //所以uref3类型为int&&
const char name[] =             //name的类型是const char[13]
 "R. N. Briggs";
 
auto arr1 = name;               //arr1的类型是const char*
auto& arr2 = name;              //arr2的类型是const char (&)[13]

void someFunc(int, double);     //someFunc是一个函数，
                                //类型为void(int, double)

auto func1 = someFunc;          //func1的类型是void (*)(int, double)
auto& func2 = someFunc;         //func2的类型是void (&)(int, double)

```

就像上述的几个示例一样auto的型别(类型)推导与模板的型别(类型)推导几乎一模一样。

# 2. 对于大括号初始化表达式的型别推导

如果要声明并初始化一个变量，在C++98里有两种方式：

``` C++

int x1 = 27;
int x2(27);

```

由于C++11也添加了用于支持统一初始化（**uniform initialization**）的语法：

``` C++

int x3 = { 27 };
int x4{ 27 };

```

因此在C++11以后有四种不同的方法对变量进行初始化，而对**auto**型别推导与模板的不同之处就在于新的初始化方式。如果把上述两段代码中的**int**类型换成**auto**，我们将会得到这样的代码：

``` C++ 伪代码

auto x1 = 27;
auto x2(27);
auto x3 = { 27 };
auto x4{ 27 };

```

此时对auto的型别推导将有所不同，使用了（**uniform initialization**）语法进行初始化的对象将被推导为std::initializer_list。这是因为关于auto的型别推导有一条特殊的不同于模板型别推导的规则：当用于auto声明变量的初始化表达式使用了同意初始化语法(用大括号括起来初始化)时，推导所得的型别类型就属于std::initializer_list。因此上述示例中的型别推导结果如下所示：

``` C++ 伪代码

auto x1 = 27;          // 型别类型是int 值是27
auto x2(27);           // 型别类型是int 值是27
auto x3 = { 27 };      // 型别类型是std::initializer_list<int> 值是 {27}
auto x4{ 27 };         // 型别类型是std::initializer_list<int> 值是 {27}

```

正是由于这一特殊的规则，将引申导致两个额外的结果：

- 模板的型别推导无法将用大括号括起来变量推导为std::initializer_list。

``` C++ 伪代码

auto x = { 11, 23, 9 }     // x的类型将被推导成为std::initializer_list<int> 

template<typename T>
void func(T param);

func({ 11, 23, 9 });       // 此时会报错，无法进行型别类型推导

template<typename T>
void func(std::initializer_list<T> param);

func({ 11, 23, 9 });       // 此时类型T将被推导成为int型，而整个传入参数的
                           // 类型为std::initializer_list<int>

```

- 当auto对大括号括起来的变量进行型别推导时，大括号内的变量类型必须完全相同否则会编译报错。

``` C++ 伪代码

auto x = { 1, 2, 3.0f };        // 此时会编译报错，原因是当进行型别推导时会先将其推导为
								// std::initializer_list<T>, 然后在对模板的型别进行推导
								// 时会发现实参中值的类型不统一进而出现编译报错

```

# 3. C++14 auto的扩展规则

当C++发展到C++14时对auto的使用范围进行了扩展，主要体现在两个方面：

- 允许使用auto来说明函数返回值需要推导。
- 允许在lambda式的形参声明中使用autor。

但这两种情况下的auto用法是在使用模板型别推导规则而非auto型别推导，这也就意味着：

``` C+++伪代码

auto CreateInitList()       // C++ 14
{
	return { 1, 2, 3 };    // 编译错误，唯一此时的auto只能使用模板型别推导规则，因此无法
						   // 完成对 { 1, 2, 3 }的型别(类型)推导
}

auto resetV = [&v] (const auto& newValue) { v = newValue; }   // C++ 14 

resetV({ 1, 2, 3 });       // 编译错误，此时auto的型别推导规则使用的是模板型别推导，因此
						   // 无法完成对 { 1, 2, 3 }的型别(类型)推导

```


# 4. 要点速记

- 在一般情况下auto的型别推导规则与模板推导是一摸一样的，但是auto型别推导会假定用大括号括起来初始化表达式代表一个std::initializer_list，但模板型别推导却不会。
- 在函数返回值或lambda的形参中使用auto，意思是使用模板型别推导而非auto型别推导。
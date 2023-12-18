---
tags:
  - cplusplus decltype
---

# 1. 初识decltype

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
- 如果表达式 ***expression*** 是一个被括号()修饰的表达式，应将()修正的***expression*** 整体作为一个表达式看待，而非简单标识符或者某个类的成员变量。此时整个被括号()修饰的表达式会被看作成一个左值，也就是说此时 ***decltype(expression)*** 表示的是 ***expression*** 的左值引用类型。

根据上述规则，我们来分析下面的代码示例：

``` C++ 伪代码

int var; 
const int&& fx(); 
struct A { double x; }; 
const A* a = new A();
int n = 0, m = 0;

decltype(var); // 使用推导规则一，var是一个不带括号的简单标识符，因此推导出的类型为var本身的类型即： int 
decltype(a->x); // 使用推导规则一，a->x是一个类的成员变量，因此推导出的类型为a->x本身的类型即： double 
decltype(fx()); // 适用推导规则二，fx()为一个函数调用因此其类型为函数返回值类型即：const int&&
decltype(m+n); // m+n表达式是两个int型数据相加的结果不能被赋值，因此是个右值，此时根据推导规则三得出其类型为： int
decltype(m=m+n); // m = m + n 表达式的结果是将m+n的结果赋给m，主体是m故可以继续给其赋值，因此该表达式是个左值表达式，根据推导规则三得出其类型为：int&
decltype((a->x)); // 此时表达式a->x是被括号()括起来的，因此不能将其看作是类的成员变量，而是将其看作一个整体。 根据推导规则四得出其类型为： double&

```

## 1.4 decltype vs auto

也许你会有这样的疑问，***decltype***与***auto***的功能都一样，都用来在编译期间进行自动型别/类型推导。那么既然已经有了***auto*** 关键字，为什么还需要 ***decltype*** 关键字呢？那是因为两者的推导规则不同，使用方式和适用场景不同，甚至有些情况下只能使用 ***decltype*** 关键字。在下文中我们将详细对比两者的区别和适用场景。

### 1.4.1 语法格式的区别

虽然***decltype***和***auto***都是C++11新增的用于型别/类型推导的关键字，但两者则用法和语法格式有着明显不同：

``` C++ 伪代码

auto varname = value;  //auto的语法格式  
decltype(exp) varname [= value];  //decltype的语法格式

```

其中，***varname*** 表示变量名，***value*** 表示赋给变量的值，***exp*** 表示一个表达式，方括号 ***[ ]*** 表示可有可无。也就是说：***auto*** 和 ***decltype*** 都会自动推导出变量 ***varname*** 的类型：

- ***auto*** 根据 `=` 右边的初始值 ***value*** 推导出变量的类型；
- ***decltype*** 根据 ***exp*** 表达式推导出变量的类型，跟 `=` 右边的 ***value*** 没有关系。

也就是说，***auto*** 要求变量必须初始化，也就是在定义变量的同时必须给它赋值；而 ***decltype*** 不要求，初始化与否都不影响变量的类型。因为 ***auto*** 是根据变量的初始值来推导出变量类型的，如果不初始化，变量的类型也就无法推导了。  
  
***auto*** 将变量的类型和初始值绑定在一起，而 ***decltype*** 将变量的类型和初始值分开；虽然 ***auto*** 的书写更加简洁，但 ***decltype*** 的使用更加灵活。

### 1.4.2 对CV限定符的处理不同

所谓CV限定符是 ***const*** 和 ***volatile*** 关键字的统称：

- ***const*** 关键字用来表示数据是只读的，也就是不能被修改；
- ***volatile*** 和 ***const*** 是相反的，它用来表示数据是可变的、易变的，目的是不让 ***CPU*** 将数据缓存到寄存器，而是从原始的内存中读取。

由于[[条款2：理解auto型别推导|auot型别推导规则]]与[[条款3：理解decltype#1.3 型别/类型推导规则|decltype型别推导规则]]的不同，这就导致在推导变量类型时，***auto*** 和 ***decltype*** 对 CV 限制符的处理是不一样的。***decltype*** 会保留 CV 限定符，而 ***auto*** 有可能会去掉 CV 限定符。

### 1.4.3 对引用的处理不同

当表达式的类型为引用时，***auto*** 和 ***decltype*** 的推导规则也不一样；***decltype*** 会保留引用类型，而 ***auto*** 会抛弃引用类型，直接推导出它的原始类型。

***auto*** 虽然在书写格式上比 ***decltype*** 简单，但是它的推导规则复杂，有时候会改变表达式的原始类型；而 ***decltype*** 比较纯粹，它一般会坚持保留原始表达式的任何类型，让推导的结果更加原汁原味，但是 ***decltype*** 总是显得比较麻烦，尤其是当表达式比较复杂时。因此虽然***auto*** ，***decltype*** 两个关键字都用来做型别/类型推导，但其适用环境和推导规则不同，我们应根据需求选用。

# 2. 理解decltype

上文已经讨论了很多关于 ***auto*** 和 ***decltype*** 的用法与区别，实际上，在C++11中，***decltype*** 最主要的用途就是用于声明函数模板，而这个函数返回类型依赖于形参类型。举个例子，假定我们写一个函数，一个形参为容器，一个形参为索引值，这个函数支持使用方括号的方式（也就是使用 ***[]*** ）访问容器中指定索引值的数据，然后在返回索引操作的结果前执行认证用户操作。函数的返回类型应该和索引操作返回的类型相同。也就是说，对一个 ***T*** 类型的容器使用 ***operator[]***  通常会返回一个 ***T&*** 对象。

使用 ***decltype*** 使得我们很容易去实现它，这是我们写的第一个版本，使用 ***decltype*** 计算返回类型，这个模板需要改良，我们把这个推迟到后面：

``` C++ 伪代码

template<typename Container, typename Index>    //可以工作，
auto authAndAccess(Container& c, Index i)       //但是需要改良
    ->decltype(c[i])
{
    authenticateUser();
    return c[i];
}

```

也许你会对 ***auto authAndAccess(Container& c, Index i)                       ->decltype(c[i])*** 的这种写法感到困惑，实际上这是C++11中的**尾置返回类型**语法。在[[条款2：理解auto型别推导#3. C++14 auto的扩展规则|C++14 auto的扩展规则]]中我们有提到：直到C++14才允许对函数返回值进行 ***auto*** 类型推导，也就是说在C++11中无法使用 ***auto*** 作为函数返回值的类型。

因此，此处 ***auto authAndAccess(Container& c, Index i)                       ->decltype(c[i])*** 的 ***auto*** 不会做任何的类型推导工作。相反的，他只是暗示使用了C++11的**尾置返回类型**语法，即在函数形参列表后面使用一个”`->`“符号指出函数的返回类型，尾置返回类型的好处是我们可以在函数返回类型中使用函数形参相关的信息。因此在C++11版本中，此处的返回值类型说明，就只可以使用 ***decltype*** (再次提醒，此处的 ***auto***  并不表示函数返回值类型是可推导类型，原因是：在C++11语法中 不支持 ***auto*** 作为函数返回值类型。)

在C++14及以后的版本中，由于[[条款2：理解auto型别推导#3. C++14 auto的扩展规则|auto的扩展规则]]，  ***auto***  允许自动推导所有的_lambda_表达式和函数。此时，对于 ***authAndAccess*** 来说这意味着在C++14标准下我们可以忽略尾置返回类型，只留下一个 ***auto*** ，如下代码所示：

``` C++ 伪代码

template<typename Container, typename Index>    //C++14版本，
auto authAndAccess(Container& c, Index i)       //不那么正确
{
    authenticateUser();
    return c[i];                     //从c[i]中推导返回类型
}

```

在C++14中直接使用 ***auto*** 定义函数返回值，乍看起来好像没有什么问题，但仔细分析就会发现上述代码并不符合我们的预期。我们原本期望的是函数返回 ***T&*** 类型的数据，但根据[[条款2：理解auto型别推导|auto型别推导规则]] ***auto*** 会忽略 ***const*** 和 引用修饰，因此实际上最终返回值的类型是 ***T*** 而非 ***T&*** 。

要想让 ***authAndAccess*** 像我们期待的那样工作，我们需要使用 ***decltype*** 类型推导来推导它的返回值，即指定 ***authAndAccess*** 应该返回一个和 ***c[i]*** 表达式类型一样的类型。C++期望在某些情况下当类型被暗示时需要使用 ***decltype*** 类型推导的规则，C++14通过使用 ***decltype(auto)*** 说明符使得这成为可能。我们第一次看见 ***decltype(auto)*** 可能觉得非常的矛盾（到底是 ***decltype*** 还是 ***auto*** ？），实际上我们可以这样解释它的意义：***auto***说明符表示这个类型将会被推导，***decltype*** 说明这个推导过程使用 ***decltype*** 的规则进行。因此我们可以这样写 ***authAndAccess*** ：

``` C++ 

template<typename Container, typename Index>    //C++14版本，
decltype(auto)                                  //可以工作，
authAndAccess(Container& c, Index i)            //但是还需要
{                                               //改良
    authenticateUser();
    return c[i];     // decltype(auto) 说明按照decltype的推导规则进行推导，也就是相当于 de
}

```
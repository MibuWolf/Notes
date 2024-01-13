---
tags:
  - cplusplus auto
---

# 1. 概述

在[[条款2：理解auto型别推导|条款2：理解auot型别推导]]中我们已经了解和讨论的`auto`的基本用法和其型别推导规则。本文我们将讨论使用`auto`带来的额外好处，以及我们推荐有限选用`auto`而非显示型别声明的原因。

# 2. 起码可以少打些字

使用`auto`的第一个也是最显而易见的好处就是：可以少打些字。如下代码示例：

``` C++

template<typename It>    // dwim（do what I wmen 做我想的函数）
void dwim(It b, It e)    // 应用范围是从b到e的所有元素，对其进行...任意操作
{
	while(b != e)
	{
		typename std::iterator_traits<It>::value_type
		currValue = *b;
		...
	}
}

```

使用`typename std::iterator_traits<It>::value_type`来表达迭代器所指涉到的元素类型写这么长的一个类型实在是毫无编码乐趣，此时使用`auto`则恰好可以将我们从这无聊的几十个字符长度的类型名中解脱出来；

``` C++

template<typename It>    // dwim（do what I wmen 做我想的函数）
void dwim(It b, It e)    // 应用范围是从b到e的所有元素，对其进行...任意操作
{
	while(b != e)
	{
		auto currValue = *b;   // 简单明了
		...
	 }
}

```

此外，我们知道使用`auto`声明的变量必须初始化，否则会导致编译错误。利用这一特点我们可以解决潜在的未初始化的风险。

一个简单的存在未初始化的风险的示例就是我们定义了一个变量但并未对其进行初始化，如下所示：

``` C++

int x;

```

如果我忘记初始化或在后续逻辑中对 `x` 赋值了，那么在使用它的时候它的值是不确定的。但严格来说，也不一定不确定也可能被初始化为 `0` 了，这取决于具体语境和环境。但总之，此时使用  `x`是存在未初始化风险的，这种风险倒不一定会引起报错或经过，但在后续运行时执行的某一刻突发一个莫名其妙的BUG将会使得问题更为复杂和难以排除。

因此使用`auto`声明的变量就很好的避免了这一潜在风险，因为如果忘记对该变量初始化则会直接导致编译报错：

``` C++

	int x;    // 存在变量未初始化的风险
	auto x1； // 编译报错，auto 声明的变量必须初始化
	auto x2 = 10; // OK 避免了变量未初始化的风险

```

# 3. 可以表示仅编译器才掌握的类型

由于 `auto`使用了型别推导，这也就意味着可以用它来表示只有编译器才掌握的型别(eg: lambda 闭包)：

``` C++

auto derefUPLess = 
[](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)
{
	return *p1 < *p2;
}

```

由于到C++14才在lambda表达式的形参中支持了`auto`，因此在C++14中可以更精简的写为：


``` C++ 14

auto derefUPLess = 
[](const auto& p1, const auto& p2)
{
	return *p1 < *p2;
}

```

可能你现在还不能感受到使用`auto derefUPLess`变量来持有这个闭包有多么深刻的意义，可能你也在想：使用C++11标准库中的 `std::function` 不是一样可以使用一个变量指向一个函数？

``` C++

std::function<bool(const std::unique_ptr<Widget>&,
				   const std::unique_ptr<Widget>&)> derefUPLess = 
[](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)
{
	return *p1 < *p2;
}

```

因为lambda表达式可以生产可调物对象，`std::function` 对象中就可以存储它。虽然看上去差不多得到了类似的效果，但其与`auto`变量还是存在很大的不同。抛开词法上罗嗦和需要制定重复的形参不谈，两者存储的变量`derefUPLess`类型就是不同的。

`auto derefUPLess`中的`derefUPLess`变量，由于其具体类型是由型别推导而来的，因此其类型与该闭包本身类型一致(lambda)。而` std::function<bool(const std::unique_ptr<Widget>&,const std::unique_ptr<Widget>&)> derefUPLess` 中的`derefUPLess`变量本质上则是一个`std::function` 类型的对象。

就内存方面而言，`auto derefUPLess`中的`derefUPLess`变量使用的内存量应该跟该闭包是一致的，而`std::function` 类型的对象则会在其构造函数中在堆上分配内存用于存储该闭包，在加上`std::function`自身数据，从结果上看`std::function`变量会比`auto`变量使用更多的内存。

在运行时就`std::function`而言，考虑到编译器的实现细节一般都会限制内联，并会产生间接的函数调用(无法通过内联的方式调用闭包)。这也就意味着，一般而言，通过`std::function`调用闭包也必然会比通过使用`auto`直接调用来的要慢。

# 4. 避免隐式类型转换问题

隐式类型转换是我们经常遇到的一个问题，例如：

``` C++

std::vector<int> v;

unsigned sz = v.size();

...

```

这是一种很常见的隐式类型转换，甚至很多时候我们自己也会写出类似的代码。实际上只有很少的人会注意到：严格来说`v.size()`返回的是`std::vecotr<int>::size_type`类型，而`std::vecotr<int>::size_type`仅仅规定成为一个无符号整型。很多人认为将将返回值定义成`unsigned`类型足够了，实际上这样写存在这一定的风险。在32位Widows上`unsigned`和`std::vecotr<int>::size_type`的尺寸是一致的，但在64位Widows上`unsigned`是32位的而`std::vecotr<int>::size_type`是64位的。这也就意味着，当这段程序移植到64位机时存在溢出的风险。

``` C++

std::vector<int> v;

auto sz = v.size(); // sz类型为std::vecotr<int>::size_type 即使移植到64位机也不会有问题

...

```

此外，用户定义不准确的一些类型导致的隐式转换，还可能存在性能问题的风险：

``` C++

std::unordered_map<std::string,int> m;

for(const std::pair<std::string,int>& p : m)
{
	...    // 在 p 上执行某些操作
}

```

这段代码，初看起来好像合情合理，但其中暗藏隐患。要想意识到哪里存在隐患，关键就是要记得`std::unordered_map`的键值部分是`const`的，因此在`std::pair`的类型不应该是`std::pair<std::string,int>`而应该是`std::pair<const std::string,int>`。意识的这问题之后我们就不难发现，在上述代码中我们将`p`声明成`const std::pair<std::string,int>`类型，这就意味着编译器会进行一次隐式类型转换。实际上`p`并不是`m`中的原始数据，而是为了进行隐式数据转换而复制出来的临时变量。

明白了这些，我们就会发现类似这样的无心之错引发的不仅是操作对象不同的风险，同时还会导致额外的复制拷贝和隐式类型转换的额外开销，而使用`auto`则可以避免此类情况：

``` C++

std::unordered_map<std::string,int> m;

for(const auto& p : m)
{
	...    // 在 p 上执行某些操作
}

```

这也恰好说明了一个问题：除非你非常确定一个变量的类型，否则显示的指定类型可能导致你既不想要，也没有想到的隐式型别转换。

# 5. 要点速记

- `auto`变量必须初始化，基本上对会导致兼容性和效率问题的类型不匹配现象免疫，还可以简化重构流程，通常也比显示指定类型要少打一些字。
- `auto`型别也可以避免一些[[条款5：优先选用auto而非显示型别声明#4. 避免隐式类型转换问题|隐式类型转换]]的问题。
- 同时`auto`型别的变量也会存在着[[条款6：当auto推导的型别不符合要求时，使用带显示型别的初始化物习惯用法|条款6]]中所描述的问题。
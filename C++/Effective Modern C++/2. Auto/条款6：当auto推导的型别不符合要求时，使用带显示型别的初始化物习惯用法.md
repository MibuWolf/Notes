---
tags:
  - cplusplus auto
---

# 1. auto可能导致不良结果

在[[条款5：优先选用auto而非显示型别声明|条款5]]中我们已经讨论了使用`auto`的一些优势和好处，但在某些情况下使用`auto`也可能会推导出不符合要求的结果。举个例子，假如我有一个函数，参数为`Widget`，返回一个`std::vector<bool>`，这个数组中的每个`bool`表示`Widget`是否提供一个独有的特性(eg: 数组中的第5个bool值表示`Widget`是否具有高优先级)。如下代码，当我们使用`auto`进行型别推导时则可能得出不符合要求的推导结果：

``` C++ 

std::vector<bool> features(const Widget& w); // 获取Widget对某些特性的支持情况

Widget w;
…
bool highPriority = features(w)[5];     //w高优先级吗？
…
processWidget(w, highPriority);         //OK 根据它的优先级处理w

auto highPriority = features(w)[5];     //推导highPriority的类型

processWidget(w,highPriority);          //未定义行为！

```

就像注释说的，第二个`processWidget`是一个未定义行为。为什么呢？答案有可能让你很惊讶，使用`auto`后`highPriority`不再是`bool`类型。虽然从概念上来说`std::vector<bool>`意味着存放`bool`，但是`std::vector<bool>`的`operator[]`不会返回容器中元素的引用（这就是`std::vector::operator[]`可返回**除了`bool`以外**的任何类型），取而代之它返回一个`std::vector<bool>::reference`的对象（一个嵌套于`std::vector<bool>`中的类）。

`std::vector<bool>::reference`之所以存在是因为`std::vector<bool>`规定了使用一个打包形式（packed form）表示它的`bool`，每个`bool`占一个`bit`。那给`std::vector`的`operator[]`带来了问题，因为`std::vector<T>`的`operator[]`应当返回一个`T&`，但是C++禁止对`bit`的引用。无法返回一个`bool&`，`std::vector<bool>`的`operator[]`返回一个**行为类似于**`bool&`的对象。为了实现返回一个`bool&`的效果，在`std::vector<bool>::reference`中实现了一个向`bool`的隐式转化。也就是说，在在第一个`features(w)[5]`的代码调用中：

``` C++

bool highPriority = features(w)[5];     //显式的声明highPriority的类型

```

`features`返回一个`std::vector<bool>`对象后再调用`operator[]`，`operator[]`将会返回一个`std::vector<bool>::reference`对象，随后会执行`std::vector<bool>::reference`类型中的隐式类型转换将`std::vector<bool>::reference`转化成为`bool`类型并最终赋值给`highPriority`。

那么在第二个`features(w)[5]`的代码调用时：

``` C++

auto highPriority = features(w)[5];     //推导highPriority的类型

```

同样的，`features`返回一个`std::vector<bool>`对象，再调用`operator[]`，`operator[]`将会返回一个`std::vector<bool>::reference`对象，但是现在这里有一点变化了，`auto`推导`highPriority`的类型为`std::vector<bool>::reference`。

而`features(w)[5]`的类型取决于`std::vector<bool>::reference`的具体实现。有一种实现方式是这样的（`std::vector<bool>::reference`）对象包含一个指向机器字（_word_）的指针，然后加上方括号中的偏移实现被引用_bit_这样的行为。

调用`features`将返回一个`std::vector<bool>`临时对象，这里把这个临时变量叫做`temp`。在`temp`上调用`operator[]`，它返回一个`std::vector<bool>::reference`对象，该对象包含一个指向存着这些_bit_s的一个数据结构中的一个_word_的指针。`highPriority`是这个`std::vector<bool>::reference`的拷贝，所以`highPriority`也包含一个指针，指向`temp`中的这个_word_，加上相应于第5个_bit_的偏移。在这个语句结束的时候`temp`将会被销毁，因为它是一个临时变量。因此`highPriority`包含一个悬置的（_dangling_）指针，故而当调用`processWidget`函数时会导致未定义行为！

# 2. 对于代理类使用显示型别的初始化物习惯用法


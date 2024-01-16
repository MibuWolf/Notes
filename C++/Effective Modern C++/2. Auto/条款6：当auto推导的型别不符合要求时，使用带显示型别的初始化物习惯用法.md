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

`std::vector<bool>::reference`本质上是一个代理类（_proxy class_）的例子，事实上，在设计模式中[[3. 结构型模式#8. 代理(Proxy)模式|代理(Proxy)模式]]是一种很常见的设计模式。我们常用的标准库中的[[4. 智能指针|智能指针]]也是用代理类实现了对原始指针的资源管理行为。

一些代理类被设计于用以对客户可见。比如`std::shared_ptr`和`std::unique_ptr`。其他的代理类则或多或少不可见，比如`std::vector<bool>::reference`就是不可见代理类的一个例子。不可见代理类多是因为设计者不希望外部使用者知道有这层代理的存在，希望使用者像使用原始数据一样的使用代理类。但是正如上文中的`std::vector<bool>::reference`示例一样，通常不可见的代理类通常不适用于`auto`。这样类型的对象的生命期通常不会设计为能活过一条语句，所以创建那样的对象你基本上就走向了违反程序库设计基本假设的道路。因此，我们应当尽量避免这种形式的代码：

``` C++

auto someVar = expression of "invisible" proxy class type;  // 既然不可见代理类的设计初衷就是让使用者无感的使用代理对象(像使用原始类型数据那样)。那么背其隐藏的不可见的代理类通常在完成类型转化后就会被销毁，使用auto会变量转化为不可见的代理类，而在其生命周期结束后访问它就很容易出现悬挂指针的情况。

```

事实上，这是一个悖论：因为不可见代理类的设计者在设计之初就是希望用户无感使用，这也就意味着我们可能根本没有意识到自己在使用一个不可见代理类。这就要求我们除了在看文档时应注意这些不可见代理类的设计说明，在看源码时也要注意。因为很少会出现源代码全都用代理对象，它们通常用于一些函数的返回类型，所以通常能从函数签名中看出它们的存在。这里有一份`std::vector<bool>::operator[]`的说明书：

``` C++

namespace std{                                  //来自于C++标准库
    template<class Allocator>
    class vector<bool, Allocator>{
    public:
        …
        class reference { … };

        reference operator[](size_type n);
        …
    };
}

```

我们知道对`std::vector<T>`使用`operator[]`通常会返回一个`T&`，在这里`operator[]`不寻常的返回类型提示你它使用了代理类。多关注你使用的接口可以暴露代理类的存在。除此之外，更为更加安全的一个解决方案是：强制使用一个不同的类型推导形式，这种方法我通常称之为显式类型初始器惯用法（_the explicitly typed initialized idiom_)。

显式类型初始器惯用法使用`auto`声明一个变量，然后对表达式强制类型转换（_cast_）得出你期望的推导结果。施加到上文中的报错代码则可修改为：

``` C++

auto highPriority = static_cast<bool>(features(w)[5]);

```

这里，`features(w)[5]`还是返回一个`std::vector<bool>::reference`对象，就像之前那样，但是这个转型使得表达式类型为`bool`，然后`auto`才被用于推导`highPriority`。在运行时，对`std::vector<bool>::operator[]`返回的`std::vector<bool>::reference`执行它支持的向`bool`的转型，在这个过程中指向`std::vector<bool>`的指针已经被解引用。这就避开了我们之前的未定义行为。然后5将被用于指向_bit_的指针，`bool`值被用于初始化`highPriority`。

这种使用显示类型初始化器的示例在实际中常常被忽略，但在细节处这往往很有用：

``` C++

double calcEpsilon();     //返回公差值

float ep = calcEpsilon();    //double到float隐式转换
auto ep1 = calcEpsilon();   // 此时ep1仍然是double
auto ep2 = static_cast<float>(calcEpsilon());   // 显示明确表示希望将类型转化为float

std::vector<float> vec;
...
int index = vec.size(); // 暗藏隐式转化将std::vecotr<int>::size_type转化成int

auto index1 = vec.size(); // 类型仍然是std::vecotr<int>::size_type

auto index2 = static_cast<int>(vec.size()); // 确表明希望转成int

```

# 3. 要点速记

- 不可见的代理类可能会使`auto`从表达式中推导出“错误的”类型
- 显式类型初始器惯用法强制`auto`推导出你想要的结果
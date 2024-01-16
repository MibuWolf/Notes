---
tags:
  - cplusplus auto
---

在[[条款5：优先选用auto而非显示型别声明|条款5]]中我们已经讨论了使用`auto`的一些优势和好处，但在某些情况下使用`auto`也可能会推导出不符合要求的结果。举个例子，假如我有一个函数，参数为`Widget`，返回一个`std::vector<bool>`，这个数组中的每个`bool`表示`Widget`是否提供一个独有的特性(eg: 数组中的第5个bool值表示`Widget`是否具有高优先级)。如下代码，当我们使用`auto`进行型别推导时则可能得出不符合要求的推导结果：

``` C++ 

std::vector<bool> features(const Widget& w); // 获取Widget对某些特性的支持情况

Widget w;
…
bool highPriority = features(w)[5];     //w高优先级吗？
…
processWidget(w, highPriority);         //根据它的优先级处理w

auto highPriority = features(w)[5];     //w高优先级吗？

processWidget(w,highPriority);          //未定义行为！

```
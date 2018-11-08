---
layout: post
title: C++中的Tagged Union
key: t20181031
tags:
  - C/C++
---

C中的union在C++中一直没有很好的替代方案，直到C++17引入了`std::variant`。

<!--more-->

`std::variant`的用法到处都查得到，我不想写废话。不过每次使用都要手写一个`class visitor`实在是太智障了，这里参照[cppreference](https://en.cppreference.com/w/cpp/utility/variant)的示例写个方便使用的Match：

{% highlight c++ linenos %}
template<typename...Ts>
using Variant = std::variant<Ts...>;

template<typename E, typename...Vs>
auto MatchVar(E &&e, Vs...vs)
{
    struct overloaded : Vs...
    {
        explicit overloaded(Vs...vss)
            : Vs(vss)...
        {

        }
    };

    return std::visit(
        overloaded(vs...),
        std::forward<E>(e));
}
{% endhighlight %}

用法是这样的：

{% highlight c++ linenos %}
Variant<int, float, std::string> tu = 5
auto v = MatchVar(tu,
    [](float f) { return f;        },
    [](int x)   { return x + 2.0f; },
    [](auto v)  { return 0.0f;     });
assert(ApproxEq(v, 7.0f));
{% endhighlight %}

有点像Rust的enum，只不过不允许手动指定Tag，只能拿类型当标记。现在稍微解析一下那一小段代码的原理。

`MatchVar`的第一个模板参数就是`Variant`的类型，无需多言。后面的变长模板参数则是针对不同类型的值做不同处理的可调用对象的类型。在内部，我定义了一个`overloaded`类，它继承了这些对象的类型，也就是说这些对象都地是类而不是函数指针之类的东西。`overloaded`在继承`Vs...`的同时也继承了它们的`operator()`方法，它的构造函数则把这一大堆`Vs`基类初始化。

最后就是用`Vs...`构造一个`overloaded`，再将它传递给`std::visit`了，没什么好说的。

使用时需要注意的有两点：
1. 每个处理对象的返回值类型必须相同，不然通不过编译；
2. 处理对象的参数类型必须覆盖`Variant`包含的所有类型，否则会爆出完全看不懂的编译错误。可以用auto参数类型的lambda来处理缺省分支，就像Rust中的`_`。

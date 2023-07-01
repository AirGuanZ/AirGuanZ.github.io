---
title: 在C++中获得当前类的类型
key: t20230702
tags:
  - C/C++
---

比我一开始以为的要困难许多，不知道有无更简单的方法。

<!--more-->

来看一个C++填空题：

```cpp
// 完善此宏，使下面的断言成立
#define DEFINE_SELF ?
struct A { DEFINE_SELF };
struct B { DEFINE_SELF };
static_assert(std::is_same_v<A::Self, A>);
static_assert(std::is_same_v<B::Self, B>);
```

一个比较直观的想法是利用`this`的类型，把`Self`提取出来：

```cpp
#define DEFINE_SELF using Self = std::remove_pointer_t<decltype(this)>;
```

让我们看看llvm-cl怎么说：

```
error : invalid use of 'this' outside of a non-static member function
```

我尝试了很多次，如用非静态成员函数做中转、从成员函数指针萃取等，都宣告失败；简单地google一下，得到的也是满屏幕的“寄”。

不过，我最终还是在某个犄角旮旯里发现了一个repo（[链接](https://github.com/MitalAshok/self_macro)），虽然没什么star，却确实地解决了我的问题，赞美MitalAshok大佬！

以下是简化过的代码实现，在我的渲染器[Rtrc](https://github.com/AirGuanZ/Rtrc/blob/main/Source/Rtrc/Utility/Struct.h)中有使用：

```cpp
template<typename T>
struct SelfTypeReader
{
    friend auto GetSelfTypeADL(SelfTypeReader);
};

template<typename T, typename U>
struct SelfTypeWriter
{
    friend auto GetSelfTypeADL(SelfTypeReader<T>) { return U{}; }
};

void GetSelfTypeADL();

template<typename T>
using SelfTypeResult = std::remove_pointer_t<decltype(GetSelfTypeADL(SelfTypeReader<T>{}))>;

#define DEFINE_SELF                                                         \
    struct SelfTypeTag{};                                                   \
    constexpr auto SelfTypeHelper() ->                                      \
        decltype(SelfTypeWriter<SelfTypeTag, decltype(this)>{}, void()) { } \
    using Self = SelfTypeResult<SelfTypeTag>;
```

之前提到过，我一直没找到怎么把`decltype(this)`利用起来，问题的核心在于没法静态地访问`this`。那么这段代码的思路就是利用auto函数返回值类型可以被延迟推导的特点，在一个可以访问`this`的地方把`decltype(this)`给记录到某个外部函数的返回类型上。

首先，`SelfTypeReader`声明了一个函数`auto GetSelfTypeADL(SelfTypeReader)`，此时我们还不知道它的返回类型是什么。接着，`DEFINE_SELF`在一个可以合法地使用`this`的地方把`decltype(this)`塞给了`SelfTypeWriter`，它内部提供了`auto GetSelfTypeADL(SelfTypeReader)`的实现，将`decltype(this)`以`GetSelfTypeADL`返回类型的方式记录下来。最后，我们借助之前定义的`SelfTypeReader`拿到需要的结果。

`SelfTypeTag`的存在只是为了确保不同对每个不同的类型，都有唯一对应的`SelfTypeReader/SelfTypeWriter`。

C++的魔法，很奇妙吧？

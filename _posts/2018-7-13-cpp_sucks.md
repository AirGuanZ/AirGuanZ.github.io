---
layout: post
title: 为什么要学Rust
key: 20180713
tags:
  - C++
  - 吐槽
---

C++真是混乱邪恶的语言。

<!--more-->

比方说，我想来个泛型四维向量`Vec4<T>`，有一些这样的代码：

```cpp
template<typename T>
AGZ_FORCE_INLINE bool ApproxEq(const Vec4<T> &lhs, const Vec4<T> &rhs, T epsilon)
{
    return ApproxEq(lhs.x, rhs.x, epsilon) &&
           ApproxEq(lhs.y, rhs.y, epsilon) &&
           ApproxEq(lhs.z, rhs.z, epsilon) &&
           ApproxEq(lhs.w, rhs.w, epsilon));
}
```

倒数第二行最后不小心多写了个括号。理想的编译器应该是这样的：

> XX行：找不到与')'配对的括号

就算不那么人性化，也应该是这样的：

> XX行：unexpected token ')'

而万恶的MSVC是这样的：

```
utils\src\math\mat4.inl(9): error C2039: “Mat4”: 不是“`global namespace'”的成员
utils\src\math\mat4.inl(10): error C2143: 语法错误: 缺少“;”(在“{”的前面)
utils\src\math\mat4.inl(10): error C2447: “{”: 缺少函数标题(是否是老式的形式表?)
utils\src\math\mat4.inl(19): error C2988: 不可识别的模板声明/定义
utils\src\math\mat4.inl(19): error C2143: 语法错误: 缺少“;”(在“<”的前面)
utils\src\math\mat4.inl(19): error C4430: 缺少类型说明符 - 假定为 int。注意: C++ 不支持默认 int
utils\src\math\mat4.inl(19): error C2374: “AGZ::Math::Mat4”: 重定义；多次初始化
utils\src\math\mat4.inl(9): note: 参见“AGZ::Math::Mat4”的声明
utils\src\math\mat4.inl(19): error C2059: 语法错误:“<”
utils\src\math\mat4.inl(19): error C2039: “Mat4”: 不是“`global namespace'”的成员
utils\src\math\mat4.inl(19): error C2039: “Data”: 不是“`global namespace'”的成员
...... 此处省略数十行和出错的地方风马牛不相及的东西 ......
utils\src\math\mat4.inl(121): error C2923: “AGZ::Math::Vec3”: 对于参数“T”，“T”不是有效的 模板 类型变量
utils\src\math\mat4.inl(122): error C2143: 语法错误: 缺少“;”(在“{”的前面)
utils\src\math\mat4.inl(122): error C2447: “{”: 缺少函数标题(是否是老式的形式表?)
utils\src\math\mat4.inl(153): error C2988: 不可识别的模板声明/定义
utils\src\math\mat4.inl(153): error C2143: 语法错误: 缺少“;”(在“<”的前面)
utils\src\math\mat4.inl(153): error C4430: 缺少类型说明符 - 假定为 int。注意: C++ 不支持默认 int
utils\src\math\mat4.inl(153): error C2374: “AGZ::Math::Mat4”: 重定义；多次初始化
utils\src\math\mat4.inl(9): note: 参见“AGZ::Math::Mat4”的声明
utils\src\math\mat4.inl(153): error C2059: 语法错误:“<”
utils\src\math\mat4.inl(153): error C2039: “Self”: 不是“`global namespace'”的成员
utils\src\math\mat4.inl(153): fatal error C1003: 错误计数超过 100；正在停止编译
```

几乎所有的错误都出现在和我写错的地方毫无关联的地方，换成gcc或者clang也好不到哪去。VS贴心地为我将错误以有问题的方式排了序，以至于编译器给出的首个错误被藏在一大票胡言乱语之间。

且不提模板元编程，这只是个普普通通、毫无技巧的模板类，就给我搞出这种幺蛾子，更别提一个未定义`operator<`的类和`std::set`一起就能搞出超过500行的错误提示了。

没有concept的模板完全就是灾难，真该好好看看隔壁Rust的泛型。

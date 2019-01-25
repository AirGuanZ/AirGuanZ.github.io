---
layout: post
title: 记一次C++模板调教过程
key: t20190123
tags:
  - C/C++
---

这篇日记名字中包含了C++模板，但并不包含什么高端的奇技淫巧或是犄角旮旯的坑。我写下它单纯是因为其内容充分显示了（我认为的）C++的一个“爽点”，即通过看起来很扭曲的模板操作实现对类型的筛选和映射。

<!--more-->

## 问题

故事的开头是这样的：我要写一大堆以常量字符串为参数的函数，且字符串的CodeUnit类型是泛型参数，据此，理想的参数类型应该是`std::basic_string_view<TChar>`，比如：

{% highlight c++ %}
template<typename TChar>
std::basic_string<TChar> trim_left(const std::basic_string_view<TChar> &str)
{
    auto it = str.begin();
    while(it != str.end())
    {
        if(!is_whitespace(*it))
            break;
        ++it;
    }
    return std::basic_string<TChar>(it, str.end());
}
{% endhighlight %}

`trim_left`会返回一个剔除了头部所有空白字符的字符串。

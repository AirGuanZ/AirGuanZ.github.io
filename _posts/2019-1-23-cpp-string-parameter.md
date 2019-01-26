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

`trim_left`会返回一个剔除了头部所有空白字符的字符串。初步测试中这个函数表现得很好，但实际使用起来就头疼了：

{% highlight c++ %}
// Compiling error
// std::string str = trim_left("  Dark Souls  ");

// Comiling error
// std::string str = trim_left(std::string("  Dark Souls  "));

// Only this works
std::string str = trim_left(std::string_view("  Dark Souls  "));
{% endhighlight %}

`const char *`和`std::string`都是可以隐式转换为`std::string_view`的；上面的`trim_left`使用的是泛型版本`std::basic_string_view`，其字符类型参数也是可以根据调用时的参数进行自动推导的。然鹅，当隐式转换和参数类型推导结合到一起的时候，问题变得极为复杂，C++就hold不住了。为了解决这个问题，我们可以特意多写两个版本的`trim_left`：

{% highlight c++ %}
template<typename TChar>
std::basic_string<TChar> trim_left(const std::basic_string<TChar> &str)
{
    return trim_left(std::basic_string_view<TChar>(str));
}

template<typename TChar>
std::basic_string<TChar> trim_left(const TChar *str)
{
    return trim_left(std::basic_string_view<TChar>(str));
}
{% endhighlight %}

别忘了我一开始提到的“要写一大堆……的函数”，如果每个字符串参数都要用这样的方式来枚举，那么要编写的函数数量将达到一个恐怖的数量——考虑有三个常量字符串参数的函数，每个参数要提供三个不同的版本，那么三个参数就要提供27个不同的签名。解决这个问题的一个办法是使用元编程（metaprogramming），用代码来生成代码，避免繁复的手工劳动。不过在这里，我考虑使用一些模板技巧来解决这个问题。

基本思路是把字符串参数的类型从`std::basic_string_view`换成一个模板参数`T`，然后用SFINAE进行约束，使得在实际调用时`T`只能具有特定的几个形式。首先把我们要的几种字符串和其他的类型区分开来：



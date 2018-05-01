---
layout: post
title: static_assert in C
key: 20180501
tags:
  - C
---

利用数组大小为负值会导致编译错误，可以在C语言中实现静态断言。

<!--more-->

{% highlight c %}
#define STATIC_ASSERT(EXPR, MSG) \
    typedef char static_assert_failed__##MSG[2 * !!(EXPR) - 1]
{% endhighlight %}

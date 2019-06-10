---
title: Alias Method
key: t20190605
tags:
  - Mathematics
---

按给定的离散分布采样是（部分领域的）编程中常见的需求，本文介绍一种$O(n)$时间预处理、$O(1)$时间获得采样点的方法，称为alias method。

<!--more-->

## 问题

输入：分布律$p_1, p_2, \ldots, p_n$，满足$p_i \in [0, 1]$且$\sum p_i = 1$

输出：某个$i \in \{ 1, 2, \ldots, n \}$，且输出$i$的概率等于$p_i$

## Native Method

简单暴力的方法，我一开始在Atrc中也是用的这个。首先求前缀和：

$$
s_k = \sum_{i=1}^k p_i~~~(k = 0, 1, \ldots, n)
$$

然后取一个$[0, 1)$上均匀采样得到的随机数$u$，在$s_{1\ldots k}$中二分搜索$m$使得：

$$
s_{m-1} \le u < s_m
$$

然后返回$m$即可。

该方法需要$O(n)$的预处理时间，每次采样则消耗$O(\log n)$。简单实现如下：

```cpp
template<typename F, typename T = int>
class bsearch_sampler_t
{
    static_assert(std::is_floating_point_v<F> && std::is_integral_v<T>);

    std::vector<F> partial_sum_;

public:

    bsearch_sampler_t() = default;

    bsearch_sampler_t(const F *prob, T n);

    void initialize(const F *prob, T n);

    bool available() const noexcept;

    void destroy();
    
    T sample(F u) const noexcept;
};

template<typename F, typename T>
bsearch_sampler_t<F, T>::bsearch_sampler_t(const F *prob, T n)
{
    initialize(prob, n);
}

template<typename F, typename T>
void bsearch_sampler_t<F, T>::initialize(const F *prob, T n)
{
    assert(n > 0);

    partial_sum_.clear();
    std::partial_sum(prob, prob + n, std::back_inserter(partial_sum_));

    F ratio = 1 / partial_sum_.back();
    for(auto &p : partial_sum_)
        p *= ratio;
}

template<typename F, typename T>
bool bsearch_sampler_t<F, T>::available() const noexcept
{
    return !partial_sum_.empty();
}

template<typename F, typename T>
void bsearch_sampler_t<F, T>::destroy()
{
    partial_sum_.clear();
}

template<typename F, typename T>
T bsearch_sampler_t<F, T>::sample(F u) const noexcept
{
    assert(available());
    assert(0 <= u && u <= 1);

    auto upper = std::upper_bound(partial_sum_.begin(), partial_sum_.end());
    if(upper == partial_sum_.end())
        return static_cast<T>(partial_sum_.size() - 1);
    return static_cast<T>(upper - partial_sum_.begin());
}
```

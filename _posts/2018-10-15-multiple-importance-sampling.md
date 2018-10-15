---
layout: post
title: 多重重要性采样
key: t20181015
tags:
  - Graphics
---

[前文](https://airguanz.github.io/2018/10/12/direct-indirect-path-tracing.html)提到当光源和物体表面材质都接近$\delta$分布时，无论是在光源上采样还是按BSDF采样都可能带来很高的方差。本文讨论一种能很好地改善该问题的技术——多重重要性采样。

<!--more-->

我们常常需要通过采样来估计某个已知部分结构特征的函数的积分（比如$f(x) = g(x)h(x)$，但函数的具体值却又严重依赖于一组额外的参数，这导致我们很难在未知参数的情况下设计出一个足够好的采样策略。举个例子，基于光线追踪的离线渲染中每条路径上辐射值的系数都是一系列因子的乘积，如果仅仅按照其中某一个来采样（如在光源上采样，或是按BSDF采样等等），那么当其他未被该采样方法考虑的因子对乘积结果的形态有显著影响时，就会产生较高的方差，亦即较低的图像质量。

多重重要性采样（multiple importance sampling，MIS）的思路是结合多种不同的采样方法，分别采样被积函数的不同部分，并将这些采样点结合起来，以达到接近于最优采样的结果。

## 多采样模型（The multi-sample model）

给定待求值的积分：

$$
\int_\Omega f(x)d\mu(x)
$$

和$n$种不同的$\Omega$上的采样策略，其概率密度函数分别记为$p_1, \ldots, p_n$。假设已经实现了以下操作：

1. 给定$x \in \Omega$，可以求出$f(x)$和$p_i(x)$。
2. 对任意$p_i$，可以按其分布生成一个样本$X$。

现假设我们已经决定了用$p_i$，来采样$n_i \le 1$个样本，于是总采样数为$N = \sum n_i$。记用第$i$个策略采样得到的第$j~(j = 1, \ldots, n_i)$个样本为$X_{i, j}$。现在的主要目标是用一种保证无偏的方式来结合所有的$X_{i, j}$。考虑给每个采样策略赋予一个权重函数$w_i~(i = 1,\ldots, n)$，则多采样估值器可以被写作：

$$
\hat F = \sum_{i=1}^n\frac 1 {n_i}\sum_{j=1}^{n_i}w_i(X_{i, j})\frac{f(X_{i, j})}{p_i(X_{i,j})}
$$

要保证$F$是无偏的，以下两个条件必须被满足：

1. $f(x) \ne 0 \Rightarrow \sum_{i=1}^nw_i(x) = 1$。
2. $p_i(x) = 0 \Rightarrow w_i(x)$。

基于这两点容易给出以下推论：对任意满足$f(x) \ne 0$的$x$，至少存在某个$i$使得$p_i(x) \ne 0$，即对$f$的每个非零点，至少要有一个$p_i$能采样到它。

**Lemma**. 若$\hat F$和$w_i$满足上面的两个无偏条件，且$n_i\le 1$，则：

$$
E[\hat F] = \int_\Omega f(x)d\mu(x)
$$

**Proof**.

$$
\begin{aligned}
E[\hat F] &= \sum_{i = 1}^n\frac 1 {n_i}\sum_{j=1}^{n_i}\int_\Omega\frac{w_i(x)f(x)}{p_i(x)}p_i(x)d\mu(x) \\
&= \int_\Omega\sum_{i=1}^nw_i(x)f(x)d\mu(x) \\
&= \int_\Omega f(x)d\mu(x)
\end{aligned}
$$

## The balance heurostic

多采样模型给出了一个巨大的无偏估值器空间以及在形式上高度统一的表示方式，我们的目标是找到合适的$w_i$以得到v具有最小方差的$\hat F$。

**Theorem**. 令：

$$
\hat w_i(x) = \frac{n_ip_i(x)}{\sum_kn_kp_k(x)}
$$

若$\hat F$使用$\hat w_i$作为权重函数，$\hat F'$是使用另一组权重函数的多采样估值器，$\mu = E[\hat F] = E[\hat F']$是被估值量，则：

$$
V[\hat F] - V[\hat F'] \le \left(\frac 1 {\min_in_i} - \frac 1 {\sum_in_i}\right)\mu^2
$$

这一定理表明$\hat w_i$无论如何都是一个相当不错的权重函数——存在比它更好的，但不会有远胜过它的。

现在把$\hat w_i$代入$\hat F$中，得到：

$$
\hat F = \sum_{i=1}^n\frac 1 {n_i}\sum_{j=1}^{n_i}\left(\frac{n_ip_i(X_{i, j})}{\sum_kn_kp_k(X_{i, j})}\right)\frac{f(X_{i,j})}{p_i(X_{i,j})}
= \frac 1 N \sum_{i=1}^n\sum_{j=1}^{n_i}\frac{f(X_{i,j})}{\sum_k\frac{n_k}{N}p_k(X_{i,j})}
$$

如果把上式进一步改写——

$$
\hat F = \frac 1 N\sum_{i=1}^n\sum_{j=1}^{n_i}\frac{f(X_{i,j})}{\hat p(X_{i,j})}~~~\text{where }\hat p(x) = \sum_{k=1}^n\frac{n_k}{N}p_k(x)
$$

这就得到了一个在形式上非常标准的蒙特卡洛估值器。$\hat p$被称为组合采样密度函数（combined sample density），是某个样本点$x$在全部$N$个采样点意义上的概率密度。

综上，我们得到了以下采样算法：

```
function BALANCE-HEURISTIC(f, p, n)
    F = 0
    N = n_1 + n_2 + ... + n_n
    for i = 1 to n
        for j = 1 to n_i
            X = TakeSample(p_i)
            hp = 0
            for k = 1 to n
                hp += n_k * p_k(X) / N
            F += f(X) / hp
    return F / N
```

## 单采样模型

像路径追踪这样有节操的算法是不会在某一条路径的某个点处采样多次的，每次出发它都一条路走到底，只不过要出发很多次罢了。因此，将上面的MIS转换为采样单个点的形式是有必要的。

设我们有$n$中采样策略，其概率密度函数分别为$p_1, \ldots, p_n$，选用哪一种策略的分布律为$c_1, \ldots, c_n~(\sum c_i = 1)$。每次采样时，先根据$c_i$选出一种采样策略，再用之前的$\hat w_i$计算估值，最后的估值器将是：

$$
\hat F = \frac{w_I(X_I)f(X_I)}{c_Ip_I(X_I)}
$$

可以证明$V[\hat F]$不大于之前按照balance heuristic策略得到的方差。

## 实现

（施工中……）

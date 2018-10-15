---
layout: post
title: 多重重要性采样
key: t20181015
tags:
  - Graphics
---

<!--more-->

我们常常需要通过采样来估计某个已知部分结构特征的函数的积分（比如$f(x) = f_1(x)f_2(x) + f_3(x)$，但函数的具体值却又严重依赖于一组额外的参数，这导致我们很难在未知参数的情况下设计出一个足够好的采样策略。举个例子，基于光线追踪的离线渲染中每条路径上辐射值的系数都是一系列因子的乘积，如果仅仅按照其中某一个来采样（如在光源上采样，或是按BSDF采样等等），那么当其他未被该采样方法考虑的因子对乘积结果的形态有显著影响时，就会产生较高的方差，亦即较低的图像质量。

本文讨论一种用于解决该问题的采样策略，称为多重重要性采样（multiple importance sampling），其思路是结合多种不同的采样方法，分别采样被积函数的不同部分，并将这些采样点结合起来，以达到接近于最优采样的结果。

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

---
layout: post
title: Monte Carlo Integration in Graphics
key: 20180820
tags:
  - Graphics
---

最近脑子都快生锈了，复习一下基本方法。

<!--more-->

## 数值求积规则

给定某定义域$\Omega$（比如$s$维立方体$[0, 1]^s$）和其上的函数$f: \Omega \to \mathbb R$，令测度函数为$d\mu(x) = dx^1\cdots dx^s$，则积分：

$$
I = \int_\Omega f(x)d\mu(x)
$$

常被用以下规则估值：

$$
\hat I = \sum_{i = 1}^N w_if(x_i)
$$

如何选择$w_i$和$x_i$？这可谓一个深不见底的大坑，常见的、针对一维情形的梯形法则、辛普森规则等大多具有形如$O(n^{-r})~(r \in \mathbb N^+)$的收敛率。在高维情形下，这一规则可以被自然地推广为：

$$
\hat I = \sum_{i_1=1}^n\sum_{i_2=1}^n\cdots\sum_{i_s=1}^nw_{i_1}w_{i_2}\cdots w_{i_s}f(x_{i_1}, x_{i_2}, \ldots, x_{i_s})
$$

推广版本的规则依然保持着$O(n^{-r})$的收敛率，但这要求$N = n^s$个采样点。换言之，若保持采样数量$N$不变，则收敛率其实降低到了$O(N^{-r/s})$，维数一上去，效率就撑不住了。

当然，不是所有高维求积规则都使用这种张量积的形式，但Bakhvalov定理已经说了，在座的各位都是垃圾：

**Bakhvalow's Theorem**. 任给$s$维求积规则，存在具有$r$阶有界连续导数的函数$f$，$f$在该规则下的误差是$O(N^{-r/s})$。

## 蒙特卡罗积分

### 估值器

给定积分：

$$
I = \int_\Omega f(x)d\mu(x)
$$

令估值器$F_N$为：

$$
F_N = \frac 1 N \sum_{i=1}^N\dfrac{f(X_i)}{p(X_i)}
$$

其中$p$是用于采样$X_i$的概率分布函数。$F_N$的方差为

$$
\begin{aligned}
V[F_N] &= V\left[\dfrac 1 N \sum_{i=1}^N\dfrac{f(X_i)}{p(X_i)}\right] \\
&= \dfrac 1 {N^2}\sum_{i=1}^NV\left[\dfrac{f(X_i)}{p(X_i)}\right] \\
&= \dfrac 1 NV\left[\dfrac{f(X)}{p(X)}\right]
\end{aligned}
$$

可见$F_N$的收敛速度是$O(\sqrt N)$。即使$V[f(X)/p(X)]$是无穷，只要$E[f(X)/p(X)]$存在，根据强大数定律$F_N$也一定会收敛（只不过要慢一些）。

### 随机变量采样

1. 已知分布函数$P$，令$U$是$[0, 1]$上的均匀随机变量，$X = P^{-1}(U)$，则$X$符合分布$P$。
2. 已知概率密度函数$p$，若存在概率密度函数$q$和常量$M$使得$p(x) \le Mq(x)$，则可先按$q$采样得到一个$X_i$，然后在$[0, 1]$内均匀采样得到一个$U_i$，若
    $$
    U_i \le p(X_i) / (Mq(X_i))
    $$
    则令采样结果为$X_i$，否则重来一遍。这样得到的采样点符合$p$。
3. Metropolis方法，这里不详述。

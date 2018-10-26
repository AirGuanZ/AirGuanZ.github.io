---
layout: post
title: Monte Carlo Integration in Graphics
key: t20180820
tags:
  - Graphics
  - Mathematics
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

## 估计量

给定积分：

$$
I = \int_\Omega f(x)d\mu(x)
$$

令估计量$F_N$为：

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

## 随机变量采样

1. 已知分布函数$P$，令$U$是$[0, 1]$上的均匀随机变量，$X = P^{-1}(U)$，则$X$符合分布$P$。
2. 已知概率密度函数$p$，若存在概率密度函数$q$和常量$M$使得$p(x) \le Mq(x)$，则可先按$q$采样得到一个$X_i$，然后在$[0, 1]$内均匀采样得到一个$U_i$，若
    $$
    U_i \le p(X_i) / (Mq(X_i))
    $$
    则令采样结果为$X_i$，否则重来一遍。这样得到的采样点符合$p$。
3. Metropolis方法，这里不详述。

设$F_N$是某个量$\mathcal Q$的一个估计量，则$F_N - \mathcal Q$称为误差，误差的期望值称为偏差：

$$
\beta[F_N] = E[F_N - \mathcal Q]
$$

偏差为0的估计量被称为是无偏的（unbiased）。另一方面，一个估计量是一致的（consistent）当且仅当误差随$N$的增大以概率1收敛到0，即：

$$
Pr\left\{\lim_{N \to \infty}F_N = \mathcal Q\right\} = 1
$$

定义均方误差$MSE[F] = E[(F - \mathcal Q)^2]$，则：

$$
\begin{aligned}
MSE[F] &= E[(F - \mathcal Q)^2] = (E[F^2] - 2E^2[F] + E^2[F]) + (E^2[F] - 2E[F]\mathcal Q + \mathcal Q^2) \\
&= E\left[(F^2 - 2FE[F] + E^2[F])\right] + E\left[(E^2[F] - 2E[F]\mathcal Q + \mathcal Q^2)\right] \\
&= E[(F - E[F])^2] + (E[F] - \mathcal Q)^2 \\
&= V[F] + \beta^2[F]
\end{aligned}
$$

对无偏估计量而言，其MSE就等于方差。于是，对某个无偏估计量$Y$，设$Y_1, Y_2, \ldots, Y_N$是其$N$个独立采样值，则：

$$
\hat V[F_N] = \dfrac 1 {N - 1}
\left\{
    \left(
        \dfrac 1 N
        \sum_{i=1}^N Y_i^2
    \right) -
    \left(
        \dfrac 1 N
        \sum_{i=1}^N Y_i
    \right)^2
\right\}
$$

是无偏估计量

$$
F_N = \dfrac 1 N\sum_{i=1}^NY_i
$$

的方差的无偏估计量。

一些常见的减小方差的方法：

### 用条件期望降维

若能计算出：

$$
f(X) = \int f(x, y)dy~~~~p(x) = \int p(x, y)dy
$$

则可作以下替换：

$$
F = \dfrac{f(X, Y)}{p(X, Y)} \Rightarrow F' = \dfrac{f(X)}{p(X)}
$$

这是因为：

$$
\begin{aligned}
E_Y\left[\dfrac{f(X, Y)}{p(X, Y)}\right]
&= \int \dfrac{f(X, y)}{p(X, y)}p(y\mid X)dy \\
&= \int \dfrac{f(X, y)}{p(X, y)}\dfrac{p(X, y)}{\int p(X, y')dy'}dy \\
&= \dfrac{f(X)}{p(X)}
\end{aligned}
$$

即$F'(X) = E_Y[F(X, Y)]$。又注意到对随机变量$G$：

$$
\begin{aligned}
V[G] &= E[G^2] - E^2[G] \\
&= E_X[E_Y[G^2]] - (E_X[E_Y[G]])^2 \\
&= E_X\left[E_Y[G^2] - E_Y^2[G]\right] + \left(E_X[E_Y^2[G]] - (E_X[E_Y[G]])^2\right) \\
&= E_X[V_Y[G]] + V_X[E_Y[G]]
\end{aligned}
$$

于是$V[F] - V[F'] = E_X[V_Y[F]]$，可见这一变换消去了$Y$带来的方差。

### 重要性采样

如果采样所使用的概率密度函数和被积函数只差一个常数因子，那么方差甚至可以是0。然而要做到这一点需要事先知道积分结果，因此不具有实践意义。尽管如此，概率密度函数和被积函数形态还是越相似越好。一种常见的方法是随便拿什么拟合一下被积函数（比方说，分段线性），然后用归一化的拟合函数作为采样的概率密度。

### Control Variates

这个翻译过来就是控制变量？总觉得怪怪的。若能找到一个形态与被积函数$f$相似但可以积出解析结果的函数$g$，则可以把原积分改写为：

$$
I = \int_\Omega f(x)d\mu(x) = \int_\Omega g(x)d\mu(x) + \int_\Omega (f(x) - g(x))d\mu(x)
$$

估计量也就变成了：

$$
F = \int_\Omega g(x)d\mu(x) + \dfrac 1 N \sum_{i=1}^N\dfrac{f(X_i) - g(X_i)}{p(X_i)}
$$

这或许能有效地降低方差：

$$
V\left[\dfrac{f(X_i) - g(X_i)}{p(X_i)}\right] \le V\left[\dfrac{f(X_i)}{p(X_i)}\right]
$$

事实上这个方法和重要性采样作用的途径相仿，因此在有了一个之后再用另一个往往效果不佳。Control variates可能在$f$严格非负的情况下带来负样本值，还有一些别的毛病，因此在图形学中没什么存在感。

### 分块采样

将积分域分成多块，每块分别估值，最后合起来。这东西不适合被积函数有奇异点或变化剧烈的情形，因而也在图形学中没什么存在感。

### Quasi-Monte Carlo

想办法把采样点均匀地散布开来，这就是耳熟能详的QMC方法。

首先定义一个衡量采样方案好坏的标准：差异度（Discrepancy）。令：

$$
\begin{aligned}
P &= \{x_1, \ldots, x_N\} \subset [0, 1]^s \\
\mathcal B^* &= \{[0, u_1]\times\ldots[0, u_s] \mid 0 \le u_i \le 1\}
\end{aligned}
$$

$P$是一堆采样点，$\mathcal B$则是一堆有个角位于原点的轴对齐盒子。在（不存在的）理想情况下，对每个体积为$\lambda(B)$的$B \in \mathcal B$，$B$中都恰好包含着$\lambda(B)N$个点。于是，可以把这个理想和现实的差距定义为差异度：

$$
D_N^*(P) = \sup_{B \in \mathcal B^*}\left|\dfrac{\#\{P\cap B\}}{N} - \lambda(B)\right|
$$

**低差异度序列**：称点列$x_1, x_2, \ldots$是一个低差异度序列，当且仅当对其任意前缀$P = \{x_1, \ldots, x_N\}$均有：

$$
D^*_N(P) = O\left(\dfrac{\log^sN}{N}\right)
$$

可以证明使用低差异度序列的蒙特卡罗积分的误差为$O(\log^sN / N)$。尽管这看起来并不是很突出（$\log^sN$可能会很大，只有在$N$极大的时候才有优势），但QMC在实践中表现非常优秀，这大概是因为图形学的被积函数对QMC而言性质良好。

Halton序列：设$i$的$b$进制展开形式为

$$
i = \sum_{k \ge 0}d_{i, k}b^k
$$

将$\phi_b(i)$定义为

$$
\phi_b(i) = \sum_{k \ge 0}d_{i, k}b^{-1-k}
$$

称序列：

$$
\{x_i\} = \{(\phi_{b_1}(i), \phi_{b_2}(i), \ldots, \phi_{b_s}(i))\}
$$

为Halton序列，其中$b_1, \ldots, b_s$两两互质（最常用的就是前$s$个素数）。特别地，若$N$在一开始就是已知的，则点集：

$$
\{x_i\} = \{(i/N, \phi_2(i), \phi_3(i), \ldots, \phi_{p_{s-1}}(i))\}
$$

被称做Hammersley点集，其差异度为$O(\log^{s-1}N/N)$。

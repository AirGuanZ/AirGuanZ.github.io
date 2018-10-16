---
layout: post
title: 多重重要性采样
key: t20181015
tags:
  - Graphics
---

<!--more-->

[前文](https://airguanz.github.io/2018/10/12/direct-indirect-path-tracing.html)提到当光源和物体表面材质都接近$\delta$分布时，无论是在光源上采样还是按BSDF采样都可能带来很高的方差。本文讨论一种能很好地改善该问题的技术——多重重要性采样。

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

现假设我们已经决定了用$p_i$，来采样$n_i \le 1$个样本，于是总采样数为$N = \sum_i n_i$。记用第$i$个策略采样得到的第$j~(j = 1, \ldots, n_i)$个样本为$X_{i, j}$。现在的主要目标是用一种保证无偏的方式来结合所有的$X_{i, j}$。考虑给每个采样策略赋予一个权重函数$w_i~(i = 1,\ldots, n)$，则多采样估值器可以被写作：

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

## 在路径追踪中的应用

像路径追踪这样有节操的算法是不会在某一条路径的某个点处采样多次的，每次出发它都一条路走到底，只不过要出发很多次罢了。因此，将上面的MIS转换为采样单个点的形式是有必要的。

考虑计算直接光照的方程（方程来源和符号含义参见[前文](https://airguanz.github.io/2018/10/12/direct-indirect-path-tracing.html)）：

$$
E(x \to \Theta) = \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)d\omega^\perp_\Phi
$$

面对此式，常用的采用策略是按$p\propto \cos\langle N_x, \Phi\rangle f_s$采样（即所谓的BSDF采样）和将采样域变换到光源表面$\mathcal M_e$上按$L_e$采样（即所谓的光源采样）。若是使用前一种方法，则当光源分布对结果影响较大时，噪点会非常明显；按后一种策略，则当BSDF描述的是光滑的表面（近乎镜面）时，图像质量也会较差。

现在考虑用重要性采样将这两种采样策略结合起来。设想以$c_1$的概率选用BSDF采样策略，以$c_2 = 1 - c_1$的概率选用光源采样策略，BSDF策略以$p_1$概率密度采样得到$\Phi$，光源策略以$p_2$概率密度采样得到$x'$。令$n_1 = n_2 = 1$，则估值器是：

$$
\begin{aligned}
\hat E(x \to \Theta) &= \frac{w_I(X_I)f_I(X_I)}{c_Ip_I(X_I)}~~~I \in \{1, 2\}\text{ sampled with }(c_1, c_2) \\
f_1(\Phi) &= f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)|N_x\cdot\Phi| \\
f_2(x') &= f_s(x' \to x \to \Theta)L_e(x' \to x)V(x', x)G(x', x) \\
G(x', x) &= \frac{|N_{x'}\cdot e_{x' \to x}||N_x\cdot e_{x \to x'}|}{|x' - x|^2}\\
w_1(\Phi) &= \frac{p_1(\Phi)}{p_1(\Phi) + p_2(\mathrm{Cast}_x(\Phi))} \\
w_2(x') &= \frac{p_2(x')}{p_1(e_{x \to x'}) + p_2(x')}
\end{aligned}
$$

稍微验证一下$E[\hat E]$，设$\mathrm{Cast}_x(\Phi)$表示从$x$出发沿$\Phi$方向的射线与全体表面$\mathcal M$的最近交点，则：

$$
\begin{aligned}
E[\hat E(x \to \Theta)] &= E\left[\frac{w_1(\Phi)f_1(\Phi)}{p_1(\Phi)}\right] + E\left[\frac{w_2(x')f_2(x')}{p_2(x')}\right] \\
&= E\left[\frac{f_1(\Phi)}{p_1(\Phi) + p_2(\mathrm{Cast}_x(\Phi))}\right] + E\left[\frac{f_2(x')}{p_1(e_{x\to x'}) + p_2(x')}\right] \\
&= \int_{\mathcal S^2}\frac{f_1(\Phi)p_1(\Phi)}{p_1(\Phi) + p_2(\mathrm{Cast}_x(\Phi))}d\omega_\Phi + \int_{\mathcal M}\frac{f_2(x')p_2(x')}{p_1(e_{x\to x'}) + p_2(x')}dA_{x'}
\end{aligned}
$$

令：

$$
\begin{aligned}
\mathcal M_{x, V} &= \{x' \in \mathcal M \mid L_e(x' \to x) \ne 0 \wedge V(x', x) = 1\}
\end{aligned}
$$

对每个$x' \in \mathcal M_{x, V}$，都一定存在一个$\Phi$使得$\mathrm{Cast}_x(\Phi) = x'$，于是:

$$
\begin{aligned}
\int_{\mathcal M}\frac{f_2(x')p_2(x')}{p_1(e_{x\to x'}) + p_2(x')}dA_{x'}
&= \int_{\mathcal M_{x, V}}\frac{f_s(x' \to x \to \Phi)L_e(x' \to x)G(x', x)p_2(\mathrm{Cast}_x(\Phi))}{p_1(e_{x \to x'}) + p_2(x')}dA_{x'} \\
&= \int_{\mathrm{Cast}_x^{-1}(\mathcal M_{x, V})}\frac{f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)p_2(\mathrm{Cast}_x(\Phi))}{p_1(\Phi) + p_2(\mathrm{Cast}_x(\Phi))}d\omega^\perp_\Phi \\
\end{aligned}
$$

若规定上式积分内的式子在$L_e(x\leftarrow \Phi)$为零时也为零，就可以上述积分的积分域扩展到$\mathcal S^2$上，得到：

$$
\int_{\mathcal M}\frac{f_2(x')p_2(x')}{p_1(e_{x\to x'}) + p_2(x')}dA_{x'} = \int_{\mathcal S^2}\frac{f_1(\Phi)p_2(\mathrm{Cast}_x(\Phi))}{p_1(\Phi) + p_2(\mathrm{Cast}_x(\Phi))}d\omega_\Phi
$$

将此式代入$E[\hat E(x \to \Theta)]$，得到：

$$
\begin{aligned}
E[\hat E(x \to \Theta)] &= \int_{\mathcal S^2}\frac{f_1(\Phi)p_1(\Phi)}{p_1(\Phi) + p_2(\mathrm{Cast}_x(\Phi))}d\omega_\Phi + \int_{\mathcal S^2}\frac{f_1(\Phi)p_2(\mathrm{Cast}_x(\Phi))}{p_1(\Phi) + p_2(\mathrm{Cast}_x(\Phi))}d\omega_\Phi \\
&= \int_{\mathcal S^2}f_1(\Phi)d\omega_\Phi = \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)|N_x\cdot\Phi|d\omega_\Phi \\
&= \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_e(x\leftarrow \Phi)d\omega^\perp_\Phi
\end{aligned}
$$

这就验证了$\hat E(x \to \Theta)$是$E(x \to \Theta)$的无偏估值器。结合[前文](https://airguanz.github.io/2018/10/12/direct-indirect-path-tracing.html)的内容，辐射值$L(x \to \Theta)$的单采样估值器是：

$$
\begin{aligned}
\hat L(x \to \Theta) &= L_e(x \to \Theta) + \hat L_s(x \to \Theta) \\
\hat L_s(x \to \Theta) &= \begin{cases}\begin{aligned}
	&\frac{f_s(\Phi \to x \to \Theta)\hat L(x \leftarrow \Phi)|N_x\cdot\Phi|}{p(\Phi)}, &f_s\text{ is specular at }x \\
	&\hat E(x \to \Theta) + \hat S(x \to \Theta), &\text{otherwise}
\end{aligned}\end{cases} \\
\hat E(x \to \Theta) &= \frac{\hat E_I(x \to \Theta)}{p(I)} \\
\hat E_1(x \to \Theta) &= \frac{p(\Phi)f_s(\Phi \to x \to \Theta)L_e(x\leftarrow\Phi)|N_x\cdot\Phi|}{p(\Phi) + p(\mathrm{Cast}_x(\Phi))} \\
\hat E_2(x \to \Theta) &= \frac{p(x')f_s(x' \to x \to \Theta)L_e(x' \to x)V(x, x')|N_{x'}\cdot e_{x' \to x}||N_x\cdot e_{x \to x'}|}{\left(p(\mathrm{Cast}_x^{-1}(x')) + p(x')\right)|x' - x|^2} \\
\hat S(x \to \Theta) &= \frac{f_s(\Phi \to x \to \Theta)\hat L_s(x \to \Theta)|N_x\cdot \Phi|}{p(\Phi)}
\end{aligned}
$$

这就是实现带有多重重要性采样的PathTracer时需要抄的公式了。当然，这只是在计算直接光照时使用了MIS，我（也许）会在以后逐步讨论更广泛的应用。

## 实现

（施工中……）

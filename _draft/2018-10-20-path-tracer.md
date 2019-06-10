---
title: 如何实现一个路径追踪器
key: t20181020
tags:
  - Graphics
  - Atrc
---

这两天在重构自己的离线渲染器[Atrc](https://github.com/AirGuanZ/Atrc)，打算把之前了解到的各种采样手段都融汇进去，在此之前让我理一理它们之间的关系，免得不小心又漏算或多算了什么。本文不是教程，只是日记。

<!--more-->

## 路径采样

原始的光线传播方程是：

$$
L(x \to \Theta) = L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega^\perp_\Phi
$$

左边的$L(x \to \Theta)$是要计算的量，右边的第一项$L_e(x \to \Theta)$直接由光源给出，第二项记为$L_s$，用单采样策略估值。

设从$x_i$处沿$\Phi$方向出发的射线与场景中物体的首个交点用$\mathrm{Cast}_{x_i}(\Phi) = x_{i+1}$表示，则：
$$
\begin{aligned}
L_e(x_i \leftarrow \Phi) &= L_e(x_{i+1} \to -\Phi) \\
L_s(x_i \leftarrow \Phi) &= L_s(x_{i+1} \to -\Phi)
\end{aligned}
$$

于是我们可以把右侧积分疯狂展开：

$$
\begin{aligned}
      & \int_{\mathcal S^2}f_s(\Phi_0 \to x_0 \to \Theta)L(x_0 \leftarrow \Phi_0)d\omega^\perp_{\Phi_0} \\
    = & \int_{\mathcal S^2}f_s(\Phi_0 \to x_0 \to \Theta)(L_e(x_0 \leftarrow \Phi_0) + L_s(x_0 \leftarrow \Phi_0))d\omega^\perp_{\Phi_0} \\
    = & \int_{\mathcal S^2}f_s(\Phi_0 \to x_0 \to \Theta)L_e(x_1 \to -\Phi_0)d\omega^\perp_{\Phi_0} + \\
    &\int_{\mathcal S^2}f_s(\Phi_0 \to x_0 \to \Theta)\left(\int_{\mathcal S^2}f_s(\Phi_1 \to x_1 \to -\Phi_0)L(x_2 \to -\Phi_1)d\omega^\perp_{\Phi_1}\right)d\omega^\perp_{\Phi_0} \\
\end{aligned}
$$



Emmm……感觉公式体型要控制不住了，稍微定义几个缩写吧：
$$
\begin{aligned}
	\Phi_{-1} &= -\Theta \\
	F(i) &= f_s(\Phi_i \to x_i \to -\Phi_{i-1}) \\
	L(i) &= L(x_{i+1} \to -\Phi_i) \\
	L_e(i) &= L_e(x_{i+1} \to -\Phi_i) \\
	L_s(i) &= L_s(x_{i+1} \to -\Phi_i) \\
	\mathcal G_i(f) &= \int_{\mathcal S^2}f(i)d\omega^\perp_{\Phi_i}
\end{aligned}
$$
根据上述定义容易得到：
$$
\begin{aligned}
	L(i) &= L_e(i) + L_s(i) \\
	L_s(i) &= \mathcal G_{i+1}(FL) \\
	\mathcal G_i(f+g) &= \mathcal G_i(f) + \mathcal G_i(g) \\
	\mathcal G^i(f) &= \mathcal G_0(F\mathcal G_1(F\mathcal G_2(...\mathcal G_n(Ff))))
\end{aligned}
$$
于是乎：
$$
\begin{aligned}
      & \int_{\mathcal S^2}f_s(\Phi_0 \to x_0 \to \Theta)L(x_0 \leftarrow \Phi_0)d\omega^\perp_{\Phi_0} \\
      &= \mathcal G_0(FL) = \mathcal G_0(F(L_e + L_s)) = \mathcal G_0(FL_e) + \mathcal G_0(FL_s) \\
      &= \mathcal G_0(FL_e) + \mathcal G_0(F(\mathcal G_1(FL))) = \mathcal G_0(FL_e) + \mathcal G_0(F(\mathcal G_1(FL_e + FL_s))) \\
      &= \mathcal G_0(FL_e) + \mathcal G_0(F\mathcal G_1(FL_e)) + \mathcal G_0(F\mathcal G_1(F\mathcal G_2(FL))) \\
      &= \cdots\cdots \\
      &= \sum_{i=0}^n\mathcal G^i(L_e) + \mathcal G^{n+1}(L) \\
      &= \sum_{i=0}^\infty\mathcal G^i(L_e)
\end{aligned}
$$
根据单采样策略：
$$
\hat{\mathcal G}^0(L_e) = \hat{\mathcal G}_0(FL_e) = \frac{F(0)L_e(0)}{p_{f_s, 0}(\Phi_0)} = \frac{f_{s, 0}(\Phi_0 \to x_0 \to -\Phi_{-1})L_e(x_1 \to -\Phi_0)|N_{x_0}, \Phi_0|}{p_{f_{s, 0}}(\Phi_0)}
$$
同理：
$$
\begin{aligned}
	\hat{\mathcal G}^n(L_e) = \prod_{i=0}^n\left(\frac{f_{s, i}(\Phi_i \to x_i \to -\Phi_{i-1})L_e(x_{i+1} \to -\Phi_i)|N_{x_i, \Phi_i}|}{p_{f_{s, i}}(\Phi_i)}\right)
\end{aligned}
$$

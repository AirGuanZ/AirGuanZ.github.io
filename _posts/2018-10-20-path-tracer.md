---
layout: post
title: 如何实现一个路径追踪器
key: t20181020
tags:
  - Graphics
  - Atrc
---

这两天在重构自己的离线渲染器[Atrc](https://github.com/AirGuanZ/Atrc)，打算把之前了解到的各种采样手段都融汇进去，在此之前让我理一理它们之间的关系，免得不小心又漏算或多算了什么。本文不是教程，只是日记。

<!--more-->

## 渲染方程，BSDF采样以及光源采样

原始的光线传播方程是：

$$
L(x \to \Theta) = L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega^\perp_\Phi
$$

左边的$L(x \to \Theta)$是要计算的量，右边的第一项$L_e(x \to \Theta)$直接由光源给出，第二项记为$L_s$，用单采样策略估值。现在我们先把右侧积分展开：

$$
\begin{aligned}
      & \int_{\mathcal S^2}f_s(\Phi \to x_0 \to \Theta)L(x_0 \leftarrow \Phi)d\omega^\perp_\Phi \\
    = & \int_{\mathcal S^2}f_s(\Phi \to x_0 \to \Theta)(L_e(x_0 \leftarrow \Phi) + L_s(x_0 \leftarrow \Phi))d\omega^\perp_\Phi
\end{aligned}
$$

设从$x_i$处沿$\Phi$方向出发的射线与场景中物体的首个交点用$\mathrm{Cast}_{x_i}(\Phi) = x_{i+1}$表示，则：

$$
\begin{aligned}
L_e(x_i \leftarrow \Phi) &= L_e(x_{i+1} \to -\Phi) \\
L_s(x_i \leftarrow \Phi) &= L_s(x_{i+1} \to -\Phi)
\end{aligned}
$$

从而我们可以疯狂展开：

$$
\begin{aligned}
      & \int_{\mathcal S^2}f_s(\Phi \to x_0 \to \Theta)L(x_0 \leftarrow \Phi)d\omega^\perp_\Phi \\
    = & \int_{\mathcal S^2}f_s(\Phi \to x_0 \to \Theta)(L_e(x_0 \leftarrow \Phi) + L_s(x_0 \leftarrow \Phi))d\omega^\perp_\Phi
\end{aligned}
$$

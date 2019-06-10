---
title: 如何写一个路径追踪器
key: t20181021
tags:
  - Graphics
  - Atrc
---

这两天在重构自己的离线渲染器[Atrc](https://github.com/AirGuanZ/Atrc)，打算把之前了解到的各种采样手段都融汇进去，在此之前让我理一理它们之间的关系，免得不小心又漏算或多算了什么。

<!--more-->

## 理论

先祭出渲染方程：
$$
L(x \to \Theta) = L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega^\perp_\Phi
$$

令：
$$
L_s(x \to \Theta) = \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega^\perp_\Phi
$$
显然有$L = L_e + L_s$，其中$L_s(x \to \Theta)$表示的是经$x$点的反射后方向为$\Theta$的辐射量。现将$L_s$定义右侧积分内部的$L$展开，同时把$f_s$拆成Specular（$f_\delta$）和Nonspecular（$f_n$）两部分：
$$
L_s(x \to \Theta) = \int_{\mathcal S^2}(f_\delta + f_n)(\Phi \to x \to \Theta)(L_e + L_s)(x \leftarrow \Phi)d\omega^\perp_\Phi
$$

考虑如何把$x \leftarrow \Phi$处理成$x' \to \Phi'$的形式。若以$x$为起点，以$\Phi$为方向的射线与场景中所有的物体表面$\mathcal M$有最近交点$\mathrm{Cast}_x(\Phi)$，则
$$
\begin{aligned}
	L_e(x \leftarrow \Phi) &= L_e(\mathrm{Cast}_x(\Phi) \to -\Phi) \\
	L_s(x \leftarrow \Phi) &= L_s(\mathrm{Cast}_x(\Phi) \to -\Phi)
\end{aligned}
$$
若$\mathrm{Cast}_x(\Phi)$不存在，即没有交点，$L_s$被认为是零，$L_e$则是那些无中生有之光源从$\Phi$方向射向$x$处的辐射量之和：
$$
\begin{aligned}
	L_e(x \leftarrow \Phi) &= \sum_{\ell \in \mathcal L}\ell(x \leftarrow \Phi) \\
	L_s(x \leftarrow \Phi) &= 0
\end{aligned}
$$
设中间量：
$$
\begin{aligned}
E(x \to \Theta) &= \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)d\omega^\perp_\Phi \\
S(x \to \Theta) &= \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L_s(x \leftarrow \Phi)d\omega^\perp_\Phi
\end{aligned}
$$
现用MIS技术来计算$E$，其估值器是：
$$
\begin{aligned}
	\hat E(x \to \Theta) &= \hat E_1(x \to \Theta) + \hat E_2(x \to \Theta) \\
	\hat E_1(x \to \Theta) &= \frac{f_s(\Phi \to x \to \Theta)L_e(x \leftarrow \Phi)|N_x\cdot\Phi|}{p_\mathrm{BSDF}(\Phi) + p_\mathrm{Light}(\Phi)} \\
	\hat E_2(x \to \Theta) &= \frac{f_s(e_{x \to x'} \to x \to \Theta)L_e(x \leftarrow e_{x \to x'})|N_x\cdot e_{x \to x'}||N_x'\cdot e_{x' \to x}|}{p_\mathrm{BSDF}(e_{x \to x'}) + p_\mathrm{Light}(e_{x \to x'})}
\end{aligned}
$$


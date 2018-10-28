---
layout: post
title: 介质中的光线传输方程
key: t20181028
tags:
  - Graphics
---

著名的渲染方程并未将传播路径中的介质考虑在内，因而无法处理雾、牛奶等物质，也不能算出我最喜欢的丁达尔效应。本文讨论带有吸收、散射等性质的介质中的光线传输方程。

<!--more-->

原始的渲染方程是：

$$
\begin{cases}
\begin{aligned}
    &L(x \to \Theta) = L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)\cos\langle N_x, \Phi\rangle d\omega_\Phi \\
    &L(x \leftarrow \Phi) = L(\mathrm{Cast}_x(\Phi) \to -\Phi)
\end{aligned}
\end{cases}
$$

而带介质的版本就装逼多了：

$$
\begin{cases}
\begin{aligned}
    &L(x \to \Theta) = L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)\cos\langle N_x, \Phi\rangle d\omega_\Phi \\
    &L(x \leftarrow \Phi) = \tau(x' \to x)L(x' \to -\Phi) + \int_0^{t_m}\tau(x + e_{x' \to x}t \to x)L_s(x + e_{x' \to x}t \to -\Phi)dt \\
    &L_s(x \to \Theta) = L_e(x \to \Theta) + \sigma_s(x \to \Theta)\int_{\mathcal S^2}\mathscr P(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega_\Phi \\
    &\tau(x' \to x) = \int_0^{t_m}\sigma_t(x + e_{x' \to x}t \to e_{x' \to x})dt \\
    &x' = \mathrm{Cast}_x(\Phi)~~~~~t_m = |x' - x|
\end{aligned}
\end{cases}
$$

其中各项的含义会在后文中逐个解释。

（施工中……）

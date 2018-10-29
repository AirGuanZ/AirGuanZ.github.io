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
    &L(x \leftarrow \Phi) = T_r(x' \to x)L(x' \to -\Phi) + \int_0^{t_m}T_r(x' + e_{x' \to x}t \to x)L_s(x' + e_{x' \to x}t \to -\Phi)dt \\
    &L_s(x \to \Theta) = L_e(x \to \Theta) + \sigma_s(x \to \Theta)\int_{\mathcal S^2}\mathscr P(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega_\Phi \\
    &T_r(x' \to x) = \int_0^{t_m}\sigma_t(x' + e_{x' \to x}t \to e_{x' \to x})dt \\
    &x' = \mathrm{Cast}_x(\Phi)~~~~~t_m = |x' - x|
\end{aligned}
\end{cases}
$$

其中各项的含义会在后文中逐个解释。

## 吸收与外散射

考虑一束方向为$\Theta$的光在空间中的$x$点因介质吸收而减弱的情形，若衰减的比例仅与这一点的介质性质以及光束本身的方向、波长有关，那么衰减量为：

$$
dL(x \to \Theta) = -\sigma_a(x \to \Theta)L(x \leftarrow -\Theta)dt
$$

其中$\sigma_a$是单位距离被吸收的光强比例，$L(x \leftarrow -\Theta)$表示这束光进入该传播距离微元的方向是$-\Theta$，这和BSDF所使用的“入射”“出射”符号习惯是一致的。容易证明，一束从$t = 0$处传播到$t = d$处的光剩下的辐射亮度比例为：

$$
\exp\left(-\int_0^d\sigma_a(x_o + te_\Theta \to \Theta)dt\right)
$$

除去吸收，介质也可能将光散射到其他方向，从而造成光强的衰减，这种散射称为外散射（out-scattering）。单位距离因外散射而损失的光强比例记作$\sigma_s$，在只考虑外散射时，有：

$$
dL(x \to \Theta) = -\sigma_s(x \to \Theta)L(x \leftarrow -\Theta)dt
$$

既然吸收和散射都会使光减弱，且具有统一的定义形式，不妨将它们合起来，称为衰减系数（Attenuation Coefficient）：

$$
\sigma_t(x \to \Theta) = \sigma_a(x \to \Theta) + \sigma_s(x \to \Theta)
$$

另外，外散射与衰减的比例被称为反照率（albedo），没错，就是BSDF里也常见到的albedo：

$$
\rho = \frac{\sigma_s}{\sigma_t}
$$

利用$\sigma_t$可以计算出一束光在穿过一段介质后还剩多少，剩余量和一开始的辐射亮度之比称为透射比，记作$T_r$：

$$
T_r(x' \to x) = \exp\left(-\int_0^d\sigma_t(x' + e_{x' \to x} \to xt)dt\right)
$$

## 自发光与内散射

介质本身可能会因为内部的一些物理/化学过程而称为光源，我们将一束光在介质中传播单位距离后因介质自发光而增加的光强记作$L_e$，即若仅考虑介质自发光，那么：

$$
dL_(x \to \Theta) = L_e(x \to \Theta)dt
$$

在外散射中被散射到其他方向的光理所当然地会增强其他方向的光的强度，这种增强叫做内散射。不同方向上的散射量也是不同的，这一分布与介质性质有关。对一束方向为$\Theta$的光，设它被散射掉的部分为$L_\Delta$，那么$\Delta$在$\Theta'$方向上的密度为：

$$
\mathscr P(\Theta \to \Theta') =
\frac
{dL_\text{out-scattering}(\Theta \to \Theta')}
{L_\Delta d\omega_{\Theta'}}
$$

$\mathscr P$被称为相函数（Phase Function）。

光在介质中传播时可能因介质的自发光和内散射而增强，我们把单位距离增加的辐射亮度记作$L_s$，即：

$$
dL(x \to \Theta) = L_s(x \to \Theta)dt
$$

根据上面对自发光和内散射的讨论，易知$L_s$可以通过下式计算：

$$
L_s(x \to \Theta) = L_e(x \to \Theta) + \sigma_s(x \to \Theta)\int_{\mathcal S^2}\mathscr P(\Phi \to \Theta)L(x \leftarrow \Phi)d\omega_\Phi
$$

## 从出射到入射

到目前为止，一开始放上的大坨公式中除了第二个（用来计算$L(x \leftarrow \Phi)$）外，其他的都已经在上面的一系列定义中给出了。设$x'$是从$x$朝$\Phi$的射线与场景的第一个交点，在普通版本的渲染方程中，辐射亮度不会在空间传播过程中衰减，因而有$L(x \leftarrow \Phi) = L(x' \to -\Phi)$；而在有介质的情况下，我们需要把传播过程中光与介质发生的所有交互都纳入计算中。首先是$L(x' \to -\Phi)$被吸收和外散射导致的衰减：

$$
T_r(x' \to x)L(x' \to -\Phi)
$$

然后在传播过程中的每一点$p$，都会因为自发光和内散射而得到一些增量，且这一部分增量在后续的传播中也要受到吸收和外散射的影响：

$$
dL_\mathrm{+} = T_r(x' + e_{x' \to x}t \to x)L_s(x' + e_{x' \to x}t \to -\Phi)dt
$$

把$dt$上得到的增量在传播路径上积分，再加上上面的$T_r(x' \to x)L(x' \to -\Phi)$，就得到了$L(x \leftarrow \Phi)$：

$$
L(x \leftarrow \Phi) = T_r(x' \to x)L(x' \to -\Phi) + \int_0^{t_m}T_r(x' + e_{x' \to x}t \to x)L_s(x' + e_{x' \to x}t \to -\Phi)dt
$$

其中$t_m$是$x'$和$x$间的距离。

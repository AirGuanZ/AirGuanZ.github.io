---
layout: post
title: 介质渲染
key: t20181028
tags:
  - Graphics
---

著名的渲染方程并未将传播路径中的介质考虑在内，因而无法处理雾、牛奶等物质，也不能算出我最喜欢的丁达尔效应。本文讨论带有吸收、散射等性质的介质中的光线传输方程，以及基于它的路径追踪算法。

<!--more-->

## 光线传输方程（LTE）

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

### 吸收与外散射

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
T_r(x' \to x) = \exp\left(-\int_0^d\sigma_t(x' + e_{x' \to x}t \to x)dt\right)
$$

### 自发光与内散射

介质本身可能会因为内部的一些物理/化学过程而称为光源，我们将一束光在介质中传播单位距离后因介质自发光而增加的光强记作$L_e$，即若仅考虑介质自发光，那么：

$$
dL(x \to \Theta) = L_e(x \to \Theta)dt
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

### 从出射到入射

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

### 相函数

本文仅考虑介质是各向同性的情况，即$\mathscr P(\Phi \to \Theta)$可以被简写为$\mathcal P(\theta)$，其中$\theta$是入射方向和出射方向间的夹角$\langle \Phi, \Theta\rangle$。根据能量守恒，$\mathscr P$必须满足：

$$
\int_{\mathcal S^2}\mathscr P(\Phi \to \Theta)d\omega_\Theta = 1
$$

据此，最为平凡的相函数——将入射光均匀散射到所有方向的相函数，是将常数1在$\mathcal S^2$上归一化后的结果：

$$
\mathscr P(\Phi \to \Theta) = \frac 1 {4\pi}
$$

1941年，Henyey & Greenstein提出了一个能够仅用一个参数就很好地拟合现实中许多物质测量结果的相函数公式，称为HG函数：

$$
\mathscr P_\mathrm{HG}(\theta) = \frac{1 - g^2}{4\pi\left(1 + g^2 + 2g\cos\theta\right)^{3/2}}
$$

其中$g$是反对称参数（asymmetry parameter），可负可正。$g$越大，介质越倾向于把更多的光散射到和$-\Phi$相近的方向。

## 路径追踪

### 框架

要将这一大坨LTE转换为可以用蒙特卡洛方法来估值的形式，还要融入MIS等采样技术，就需要把LTE细致地分解开来。首先我们约定：用$x$表示表面（其实也就是不同介质的分界面）上的位置，用$p$表示介质内部的位置，于是$L(x \leftarrow \Phi)$和$L(p \leftarrow \Phi)$就有了完全不同的含义。在这一符号约定下，LTE变成了：

$$
\begin{cases}
\begin{aligned}
    L(x \to \Theta) &= L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega^\perp_\Phi \\
    L(x \leftarrow \Phi) & = \begin{cases}\begin{aligned}
        &T_r(x' \to x)L(x' \to -\Phi) + \int_{x'x}T_r(p \to x)L_s(p \to -\Phi)dl_p, &x'\text{ exists} \\
        &\int_{\mathrm{ray}(x, \Phi)}T_r(p \to x)L_s(p \to -\Phi)dl_p, &\text{otherwise}
    \end{aligned}\end{cases} \\
    L(p \leftarrow \Phi) & = \begin{cases}\begin{aligned}
        &T_r(x' \to p)L(x' \to -\Phi) + \int_{x'p}T_r(p' \to p)L_s(p' \to -\Phi)dl_{p'}, &x'\text{ exists} \\
        &\int_{\mathrm{ray}(p, \Phi)}T_r(p' \to p)L_s(p' \to -\Phi)dl_{p'}, &\text{otherwise}
    \end{aligned}\end{cases} \\
    L_s(p \to \Theta) &= L_e(p \to \Theta) + \sigma_s(p \to \Theta)\int_{\mathcal S^2}\mathscr P(\Phi \to p \to \Theta)L(p \leftarrow \Phi)d\omega_\Phi \\
    T_r(a \to b) &= \int_{ab}\sigma_t(p \to e_{a \to b})dl_p
\end{aligned}
\end{cases}
$$

此时，我们将表面上某点的出射光分解为自发光和散射光：

$$
L(x \to \Theta) = L_e(x \to \Theta) + L_2(x \to \Theta)
$$

其中$L_2$表示$x$点散射其他入射光而产生的出射光，其定义为：

$$
L_2(x \to \Theta) = \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)d\omega^\perp_\Phi
$$

由于介质的参与，$L(x \leftarrow \Phi)$的计算相当不平凡。为了书写简便，后文不再列出$x'$不存在时的情形（即LTE中的“$\mathrm{otherwise}$”）。现令：

$$
\begin{aligned}
    D_1(x \leftarrow \Phi) &= T_r(x' \to x)L_e(x' \to -\Phi) \\
    D_2(x \leftarrow \Phi) &= T_r(x' \to x)L_2(x' \to -\Phi) + \int_{x'x}T_r(p \to x)L_s(p \to -\Phi)dl_p
\end{aligned}
$$

则：

$$
L(x \leftarrow \Phi) = D_1(x \leftarrow \Phi) + D_2(x \leftarrow \Phi)
$$

于是$L_2$也可以对应地分解为两项：

$$
\begin{aligned}
    L_2(x \to \Theta) &= E(x \to \Theta) + S(x \to \Theta) \\
    E(x \to \Theta) &= \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)D_1(x \leftarrow \Phi)d\omega^\perp_\Phi \\
    S(x \to \Theta) &= \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)D_2(x \leftarrow \Phi)d\omega^\perp_\Phi \\
\end{aligned}
$$

对应的估计量是：

$$
\hat L_2(x \to \Theta) = \hat E(x \to \Theta) + \hat S(x \to \Theta)
$$

$E$是所谓的直接照明项，和[前文]({{site.url}}/2018/10/15/multiple-importance-sampling.html)一样使用MIS技术来采样，只不过计算辐射亮度时都要乘上一个透射比罢了。$S$的采样则要复杂一些，因为它的里面出现了介质发光和内散射产生的增益。

### 间接照明

在计算$S(x \to \Theta)$时，设想我们用概率密度$p_s$进行BSDF采样，选取了入射方向$\Phi$，且$x' = \mathrm{Cast}_x(\Phi)$存在（$x'$不存在的情形更加简单，因此这里略过），那么$S$的估计量是：

$$
\hat S(x \to \Theta) = \frac{\hat D_2(x \leftarrow \Phi)f_s(\Phi \to x \to \Theta)\cos\langle N_x, \Phi\rangle}{p_s(\Phi)}
$$

要计算$\hat D_2$，我们首先需要在线段$x'x$上以概率密度$p_{x'x}$采样点$p$，然后根据$D_2$的定义：

$$
\hat D_2(x \leftarrow \Phi) = \hat T_r(x' \to x)\hat L_2(x' \to -\Phi) + \frac{\hat T_r(p \to x)\hat L_s(p \to -\Phi)}{p_{x'x}(p)}
$$

$\hat L_s$的计算又涉及到对相函数$\mathscr P$进行重要性采样。设概率密度函数为$p_{\mathscr P}$，采样得到的方向为$\Phi$，则：

$$
\hat L_s(p \to \Theta) = L_e(p \to \Theta) + \sigma_s(p \to \Theta)\frac{\mathscr P(\Phi \to p \to \Theta)\hat L(p \leftarrow \Phi)}{p_{\mathscr P}(\Phi)}
$$

$L(p \leftarrow \Phi)$和$L(x \leftarrow \Phi)$在形式上一样，其估计量的计算也完全相同，这里就不再赘述了。

### 透射比

根据定义，有：

$$
T_r(x' \to x) = \exp\left(-\int_0^d\sigma_t(x' + e_{x' \to x}t \to x)dt\right)
$$

因此，我们可以在$[0, d]$中以概率密度函数$p_T$选取$N_T$个$t$，然后令：

$$
\hat T_r(x' \to x) = \exp\left(-\frac 1 {N_T}\sum_{i=1}^{N_T}\frac{\sigma_t(x' + e_{x' \to x}t_i \to x)}{p_T(t_i)}\right)
$$

路径追踪算法通常把$N_T$设置为1。如果是在均匀介质中，那么$\sigma_t$处处相等，可以这样计算：

$$
\hat T_r(x' \to x) = \exp(-\sigma_td)
$$

### 小结

总结上面的讨论以及过去MIS的文章（用来计算$\hat E）$，就得到了以下的一系列估计量，其中涉及到采样的地方都将采样的概率密度和随机变量标注在式子后方：

$$
\begin{cases}
\begin{aligned}
    \hat L(x \to \Theta) &= L_e(x \to \Theta) + \hat L_2(x \to \Theta) \\
    \hat L_2(x \to \Theta) &= \hat E(x \to \Theta) + \hat S(x \to \Theta) \\
    \hat E(x \to \Theta) &= \hat E_1(x \to \Theta) + \hat E_2(x \to \Theta) \\
    \hat E_1(x \to \Theta) &= \frac{L_e(x' \to -\Phi)f_s(\Phi \to x \to \Theta)\cos\langle N_x, \Phi\rangle}{p_s(\Phi) + p_\ell(x')} &[p_s, \Phi] \\
    \hat E_2(x \to \Theta) &= \frac{L_e(x' \to e_{x' \to x})f_s(e_{x \to x'} \to x \to \Theta)V(x', x)G(x', x)}{(p_s(e_{x \to x'}) + p_\ell(x'))} &[p_\ell, x'] \\
    G(x', x) &= \frac{\cos\langle N_x, e_{x \to x'}\rangle\cos\langle N_{x'}, e_{x' \to x}\rangle}{|x' - x|^2} \\
    \hat S(x \to \Theta) &= \frac{\hat D_2(x \leftarrow \Phi)f_s(\Phi \to x \to \Theta)\cos\langle N_x, \Phi\rangle}{p_s(\Phi)} &[p_s, \Phi] \\
    \hat D_2(x \leftarrow \Phi) &= \hat T_r(x' \to x)\hat L_2(x' \to -\Phi) + \frac{\hat T_r(p \to x)\hat L_s(p \to -\Phi)}{p_{x'x}(p)} &[p_{x'x}, p] \\
    \hat L_s(p \to \Theta) &= L_e(p \to \Theta) + \sigma_s(p \to \Theta)\frac{\mathscr P(\Phi \to p \to \Theta)\hat L(p \leftarrow \Phi)}{p_{\mathscr P}(\Phi)} &[p_{\mathscr P}, \Phi] \\
    \hat L(x \leftarrow \Phi) &= \hat T_r(x' \to x)L_e(x' \to -\Phi) + \hat D_2(x \leftarrow \Phi) \\
    \hat L(p \leftarrow \Phi) &= \text{ same as }\hat L(x \leftarrow \Phi) \\
    \hat T_r(a \to b) &= \exp\left(-\frac{\sigma_t(p \to e_{a \to b})}{p_T(p)}\right) &[p_T, p]
\end{aligned}
\end{cases}
$$

## 实现

实现见[Atrc](https://github.com/AirGuanZ/Atrc/tree/master/Source/Atrc/Integrator)中的VolumetricPathTracer。随便画了点东西：

![PICTURE]({{site.url}}/postpics/Atrc/2018_11_06_Vol.png)

此外，我现在有点不知道该如何处理室外场景中雾和太阳光这样的光源的关系，后面再慢慢探究吧。

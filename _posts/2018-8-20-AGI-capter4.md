---
layout: post
title: 读书笔记：Strategies for computing light transport
key: 20180820
tags:
  - Graphics
---

Advanced Global Illuminations第四章内容。

<!--more-->

## 渲染方程

简要地复习一下渲染方程，首先是原版：

$$
L(x \to \Theta) = L_e(x \to \Theta) + \int_{\Omega_x}f_r(x, \Psi \leftrightarrow \Theta)L(x \leftarrow \Psi)\cos(N_x, \Psi)d\omega_\Psi
$$

这是个第二类Fredholm积分方程，其中$L(x \to \Theta)$表示从位置$x$向$\Theta$发射的辐射亮度；$L_e$是自发光项；$N_x$是$x$处法线；$\Omega_x$是以$N_x$为中心的半球立体角；$f_r(x, \Psi \leftrightarrow \Theta)$是BRDF，其形式化定义为：

$$
f_r(x, \Psi \to \Theta) =
\dfrac
{dL(x \to \Theta)}
{L(x \leftarrow \Psi)\cos(N_x, \Psi)d\omega_\Psi}
$$

通常，BRDF函数的两个方向参数是对称的，所以也可以写做$\Psi \leftarrow \Theta$甚至$\Psi \leftrightarrow \Theta$。

令$r(x, \Psi)$表示从$x$处向$\Psi$方向发射射线所获得的最近交点，$y = r(x, \Psi)$，则有：

$$
L(x \to \Theta) = L_e(x \to \Theta) + \int_{\Omega_x}f_r(x, \Psi \leftrightarrow \Theta)L(y \to -\Psi)\cos(N_x, \Psi)d\omega_\Psi
$$

上述积分是在$x$的半球立体角上进行的，也可以把它改写成在场景中所有表面上的形式：

$$
L(x \to \Theta) = L_e(x \to \Theta) + \int_Af_r(x, \Psi \leftrightarrow \Theta)L(y \to \overrightarrow{yx})V(x, y)G(x, y)dA_y
$$

其中$G(x, y) = \cos(N_x, \Psi)\cos(N_y, -\Psi)/r^2_{xy}$为场景中任两个表面点间的几何因子。

这些公式都是出射的，也可以写出入射形式：

$$
L(x \leftarrow \Theta) = L_e(x \leftarrow \Theta) + \int_{\Omega_y}f_r(y, \Psi \leftrightarrow -\Theta)L(y \leftarrow \Psi)\cos(N_y, \Psi)d\omega_\Psi
$$

$$
L(x \leftarrow \Theta) = L_e(x \leftarrow y) + \int_Af_r(y, \Psi \leftrightarrow \overrightarrow{yz})L(y \leftarrow \overrightarrow{yz})V(y, z)G(y, z)dA_z
$$

## 重要性函数

由于无法无限精确地表示场景中所有点朝所有方向的辐射亮度，几乎所有的照明算法都是在计算一些点集朝一些方向的辐射亮度均值。设$S = A_s \times \Omega_s$表示希望纳入计算的、局部的表面点集和方向，则：

$$
\Phi(S) = \int_{A_s}\int_{\Omega_s}L(x \to \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x
$$

若定义：

$$
W_e(x \leftarrow \Theta) = \begin{cases}
1, (x, \Theta) \in S \\
0, (x, \Theta) \notin S
\end{cases}
$$

则$S$上的平均辐射亮度可以被表示为：

$$
L_\text{avg} = \dfrac
{\int_A\int_\Omega L(x \to \Theta)W_e(x \leftarrow \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x}
{\int_A\int_\Omega W_e(x \leftarrow \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x}
$$

这个Importance函数的意义是什么呢？给定场景中某个感兴趣的集合$S$，若将某单个的$L(x \to \Theta)$放置在场景的某个表面点上，它会对$\Phi(S)$产生的贡献可以用来衡量$x \to \Theta$在$S$中的“重要程度”。形式化地，可以将$W(x \leftarrow \Theta)$定义如下：

$$
\begin{aligned}
& W_e(x \leftarrow \Theta) = \begin{cases}
1, (x, \Theta) \in S \\
0, (x, \Theta) \notin S
\end{cases} \\
& W(x \leftarrow \Theta) = W_e(x \leftarrow \Theta) + \int_{\Omega_z}f_r(z, \Psi \leftrightarrow -\Theta)W(z \leftarrow \Psi)\cos(N_{r(x, \Theta)}, \Psi)d\omega_\Psi
\end{aligned}
$$

其中$z = r(x, \Theta)$。重要性函数值仅与场景的几何和材质特征有关，传递方式与辐射亮度的传播相似。

由于重要性函数的定义形式和入射版本的渲染方程一模一样，于是可以对照着定义一个出射版（可以理解为某种“重要性发射器”）。令$W(x \to \Theta) = W(r(x, \Theta) \leftarrow -\Theta)$，容易证明：

$$
W(x \to \Theta) = W_e(x \to \Theta) + \int_{\Omega_x}f_r(x, \Psi \leftrightarrow \Theta)W(x \leftarrow \Psi)\cos(N_x, \Psi)d\omega_\Psi
$$

有了重要性函数，在实际计算辐射通量的时候，间接传播所作的贡献都被重要性函数纳入怀中了，因此有下面两个等价的公式：

$$
\begin{aligned}
&\Phi(S) = \int_A\int_{\Omega_x}L_e(x \to \Theta)W(x \leftarrow \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x \\
&\Phi(S) = \int_A\int_{\Omega_x}L_e(x \leftarrow \Theta)W(x \to \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x \\
&\Phi(S) = \int_A\int_{\Omega_x}L(x \to \Theta)W_e(x \leftarrow \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x \\
&\Phi(S) = \int_A\int_{\Omega_x}L(x \leftarrow \Theta)W_e(x \to \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x
\end{aligned}
$$

这样一来，我们就有了两种计算全局照明的思路：从$S$出发按照渲染方程“寻找”光源，或是从光源出发，计算光源对$S$的重要性函数。

## 伴随方程

首先引入两个线形算子：

$$
\begin{aligned}
& \mathcal TL(x \to \Theta) = \int_{\Omega_x}f_r(x, \Psi \leftrightarrow \Theta)L(r(x, \Theta) \to -\Psi)\cos(N_x, \Psi)d\omega_\Psi \\
& \mathcal QW(x \leftarrow \Theta) = \int_{\Omega_{r(x, \Theta)}}f_r(r(x, \Theta), \Psi \leftrightarrow -\Theta)W(r(x, \Theta) \leftarrow \Psi)\cos(N_{r(x, \Theta)}, \Psi)d\omega_\Psi
\end{aligned}
$$

设$L^\rightarrow$表示出射辐射亮度，$L^\leftarrow$表示入射辐射亮度，$W^\rightarrow, W^\leftarrow$类似，则有：

$$
\begin{aligned}
& L^\rightarrow = L^\rightarrow_e + \mathcal TL^\rightarrow \\
& L^\leftarrow = L^\leftarrow_e + \mathcal QL^\leftarrow \\
& W^\rightarrow = W^\rightarrow_e + \mathcal TW^\rightarrow \\
& W^\leftarrow = W^\leftarrow_e + \mathcal QW^\leftarrow
\end{aligned}
$$

现定义$A \times \Omega$上的出射函数和入射函数间的内积为：

$$
\langle F^\rightarrow, G^\leftarrow \rangle = \int_A\int_\Omega F(x \to \Theta)G(x \leftarrow \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x
$$

则：

$$
\begin{aligned}
& \Phi(S) = \langle L^\rightarrow, W_e^\leftarrow \rangle \\
& \Phi(S) = \langle L_e^\rightarrow, W^\leftarrow \rangle \\
& \Phi(S) = \langle L^\leftarrow, W_e^\rightarrow \rangle \\
& \Phi(S) = \langle L_e^\leftarrow, W^\rightarrow \rangle
\end{aligned}
$$

容易验证$\mathcal T = \mathcal Q^*$，即$\mathcal T$是$\mathcal Q$的伴随算子，于是乎：

$$
\begin{aligned}
& L^\leftarrow = L_e^\leftarrow + \mathcal T^*L^\leftarrow \\
& W^\leftarrow = W_e^\leftarrow + \mathcal T^*W^\leftarrow
\end{aligned}
$$

## 全局反射分布

一切$L$都来自于$L_e$，这一关系由辐射亮度的传输方程给出。但是传输方程给出的式子是递归的，接下来给出一个非递归版本，称为全局反射分布函数（GRDF），记作$G_r(x\leftarrow\Theta, y\rightarrow\Psi)$。GRDF描述了场景中任意两处之间的光照传输属性，它满足：

$$
L(y \to \Psi) = \int_A\int_{\Omega_x}L_e(x \to \Theta)G_r(x \leftarrow \Theta, y \to \Psi)\cos(N_x, \Theta)d\omega_\Theta dA_x
$$

上式两边微分可得：

$$
G_r(x \leftarrow \Theta, y \rightarrow \Psi) =
\dfrac
{d^2L(y \to \Psi)}
{L_e(x \to \Theta)\cos(N_x, \Theta)d\omega_\Theta dA_x}
$$

事实上，BRDF可以被看作GRDF的一种特例。

有了GRDF过后，辐射通量可以这样计算：

$$
\Phi(S) = \int_A\int_{\Omega_x}\int_A\int_{\Omega_y}L_e(x \to \Omega)G_r(x \leftarrow \Theta, y \to \Psi)W_e(y \leftarrow \Psi)\times\cos(N_x, \Theta)\cos(N_y, \Psi)d\omega_\Psi dA_yd\omega_\Theta dA_x
$$

GRDF的好处在于，它提供的光照传输描述仅仅依赖于场景几何与材质。尽管重要性函数也具有这一特征，但它依赖于特定的$S$。话虽如此，由于参数空间维度太高，很难直接计算GRDF，而多采用蒙特卡罗之类的方法来估计。著名的双向路径追踪算法就可以被认为是基于GRDF的。

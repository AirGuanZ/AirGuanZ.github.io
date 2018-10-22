---
layout: post
title: 关于材质模型
key: t20181022
tags:
  - Graphics
  - Atrc
---

在实现了Multiple Importance Sampling后，我发现[Atrc](https://github.com/AirGuanZ/Atrc)居然没有Glossy类型的材质，以至于我很难构造一个能够展现MIS技术优越性的场景。因此，下一步个小目标就是实现一个能用的材质系统了。

<!--more-->

## BSDF

BSDF的全称是Bidirectional Scattering Distribution Function，即双向散射分布函数。对场景中的某个物体表面的一点，设想从$\Phi$方向射来一束光，这束光必然被反射/折射向空间中的各个方向，而每个方向所分得的“比例”就是由BSDF描述的。

本文使用符号$f_s(\Phi \to x \to \Theta)$表示入射方向为$\Phi$时，$x$处的材质将这些辐射量反射到$\Theta$方向的比例。严格地说，是出射辐射和入射照度之比：

$$
f_s(\Phi \to x \to \Theta) = \frac{dL(x \to \Theta)}{dE(x \leftarrow \Phi)}
$$

根据 Lambertian定律和$L$与$E$间的关系，有：

$$
dE(x\leftarrow\Phi) = L(x\leftarrow\Phi)\cos\langle N_x, \Phi\rangle d\omega_\Phi
$$

其中$N_x$是$x$处的表面法线。将$dL(x \to \Theta)$表示成$f_s(\Phi \to x \to \Theta)dE(x \leftarrow \Phi)$，再在各个可能的$\omega_\Phi$上积分，就得到了：

$$
L_s(x \to \Theta) = \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)|N_x\cdot\Phi|d\omega_\Phi
$$

$L_s(x \to \Theta)$就是从其他地方照射到$x$处，再散射向$\Theta$方向的辐射量了。如果再加上$x$处本身的自发光，我们甚至可以立刻导出渲染方程：

$$
L(x \to \Theta) = L_e(x \to \Theta) + \int_{\mathcal S^2}f_s(\Phi \to x \to \Theta)L(x \leftarrow \Phi)|N_x\cdot\Phi|d\omega_\Phi
$$

考虑到大部分材料都是不透明的，其BSDF的非零范围仅限于表面法线方向的半立体角$\mathcal H^2$而已，我们可以把BSDF定义域中的方向限制到$\mathcal H^2$上，得到的函数称为Bidirection Reflectance Distribution Function，即双向反射分布函数，简记为BRDF，用符号$f_r$表示。类似的定义还有描述折射的BTDF $f_t$等，这里不再赘述。

## Fresnel Formula

一束光照射到一个平滑表面时，有多大的比例被反射、折射或是吸收？我们的第一反应可能是给个反射系数/折射系数作为材质参数。但这个问题是可以从Maxwell方程组解出来的（好像还在《电磁场与波》这门课上解过……），若是凭感觉乱写，在真实感图形绘制中反倒落了下乘。

给定两种绝缘体介质，设光从折射率为$\eta_i$的一侧照射到两种介质的（平滑）分界面上，另一侧介质的折射率为$\eta_t$，则若忽略入射和出射光可能带有的极化特征，反射光的占比为：

$$
\begin{aligned}
	F_r &= \frac 1 2 (r^2_\parallel + r^2_\perp) \\
	r_\parallel &= \frac{\eta_t\cos\theta_i - \eta_i\cos\theta_t}{\eta_t\cos\theta_i + \eta_i\cos\theta_t} \\
	r_\perp &= \frac{\eta_i\cos\theta_i - \eta_t\cos\theta_t}{\eta_i\cos\theta_i + \eta_t\cos\theta_t}
\end{aligned}
$$

其中$\theta_i$和$\theta_t$分别是入射光和折射光与分界面在各自一侧的法线的夹角。

导体的Fresnel系数计算依赖于一个更加一般的公式，其中导体本身的“折射率”以$\eta_t + ik'$的复数形式给出，$k'$代表了材料对入射光的吸收率。设$\eta = \eta_t / \eta_i, k = k' / \eta_i$，入射角度为$\theta​$，则：

$$
\begin{aligned}
	F_r &= \frac 1 2(r^2_\parallel + r^2_\perp) \\
	r_\parallel &= r_\perp\frac{\cos^2\theta(a^2 + b^2) - 2a\cos\theta\sin^2\theta + \sin^4\theta}{\cos^2\theta(a^2 + b^2) + 2a\cos\theta\sin^2\theta + \sin^4\theta} \\
	r_\perp &= \frac{a^2 + b^2 - 2a\cos\theta + \cos^2\theta}{a^2 + b^2 + 2a\cos\theta + \cos^2\theta} \\
	a^2 + b^2 &= \sqrt{(\eta^2 - k^2 - \sin^2\theta)^2 + 4\eta^2k^2}
\end{aligned}
$$

（施工中……）

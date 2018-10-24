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

根据Lambertian定律和$L$与$E$间的关系，有：

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

## Perfect Diffuse Reflection

完美漫反射表面会将入射光均匀地反射到法线方向半立体角上的每一个方向，这样的材质在真实世界中应该是找不到的，但却是一种非常基本的“分量”，绝大部分材质的BRDF中都或多或少地包含了漫反射成分。

将“均匀反射到各个方向”形式化，就得到了下述公式：

$$
f_r(\Phi \to x \to \Theta) = \mathrm{CONST\_VALUE}~~~~\Phi, \Theta \in \mathcal H^2
$$

也就是说$f_r$是个常值函数。现在假设我希望有一个表面，它反射入射光的能量比例为$a$，且反射到各方向的辐射都是相同的，则$f_r$必然具有形式：

$$
f_r(\Phi \to x \to \Theta) = ca
$$

如何计算这个$c$呢？设入射照度为$I$，则出射照度为$E = aI$，据此：

$$
\begin{aligned}
E &= aI \\
E &= \int_{\mathcal H^2}\left(\int_{\mathcal H^2}caL(x\leftarrow\Phi)d\omega^\perp_\Phi\right)d\omega^\perp_\Theta \\
I &= \int_{\mathcal H^2}L(x \leftarrow \Phi)d\omega^\perp_\Phi
\end{aligned}
$$

联立解得

$$
c = \frac 1 \pi
$$

因此理想漫反射表面的BRDF为：

$$
f_r(\Phi \to x \to \Theta) = \frac a \pi ~~~~\Phi, \Theta \in \mathcal H^2
$$

随便画个图意思一下：

![PathTracerExWithSpecularSampling]({{site.url}}/postpics/Atrc/2018-10-23-perfect-diffuse-reflection.png)

## Fresnel Formula

一束光照射到一个平滑表面时，有多大的比例被反射、折射或是吸收？我们的第一反应可能是给个反射系数/折射系数作为材质参数。但这个问题是可以从Maxwell方程组解出来的（好像还在《电磁场与波》这门课上解过……），若是凭感觉乱写，在真实感图形绘制中反倒落了下乘。

给定两种绝缘体介质，设光从折射率为$\eta_i$的一侧照射到两种介质的（平滑）分界面上，另一侧介质的折射率为$\eta_t$，若忽略入射和出射光可能带有的极化特征，反射光的占比为：

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
	a^2 + b^2 &= \sqrt{(\eta^2 - k^2 - \sin^2\theta)^2 + 4\eta^2k^2} \\
	a &= \sqrt{\frac 1 2\left(a^2 + b^2 + \eta^2 - k^2 - \sin^2\theta\right)}
\end{aligned}
$$

## Perfect Specular Reflection

带有很多镜面反射的场景很适合用来展示光线追踪技术的威力，光栅化（在我的知识范围内）算法则无能为力。因此我以很高的优先度实现镜面反射。

一般材质会将从某个方向来的光反射到许多方向去，因此这些方向的反射量可以用分布函数来描述；而镜面反射将某个方向$\Phi$的入射光全部反射到另一个特定的方向$\Theta$上去，如果依然用分布函数来描述，会形成在$\Theta$上值为无限大，而在其他方向上值为零的尴尬情境。这样的情境下，我们称反射分布是一个奇异分布，并用$\delta$函数来表示它。

此外，在使用BRDF来计算辐射值时，都会根据Lambertian定律在$f_r$旁边乘上$\cos\langle N_x, \Phi\rangle$因子，这对理想镜面反射来说反而是画蛇添足了，因此在$f_r$中我们得除一下它。最终$f_r$的表达式是：

$$
f_r(\Phi \to x \to \Theta) = F_r\frac{\delta(\Theta - \mathrm{Ref}_x(\Phi))}{\cos\langle N_x, \Phi\rangle}
$$

其中$F_r$是之前提到的Fresnel项，$\mathrm{Ref}_x(\Phi)$是根据法线$N_x$和入射方向$\Phi$计算镜面反射出射方向的函数，用初中物理学过的“入射角等于反射角”就能推出来（假设法线和方向向量都是归一化的）：

$$
\mathrm{Ref}_x(\Phi) = 2(\Phi\cdot N_x)N_x - \Phi
$$

材质效果如下：

![PathTracerExWithSpecularSampling]({{site.url}}/postpics/Atrc/23_2018_10_23_IdealMirror.png)

## Perfect Specular Transmission

有了理想反射自然就要有理想折射，这两者凑在一起才能制作出玻璃钻石之类的特效。然而折射的天然特性使得它比反射更困难一些——

1. 折射角和介质折射率有关，服从Snell定律
2. 要考虑全反射
3. 折射角的变化会“压缩”或“扩张”立体角微元，使得辐射值等比例地发生变化

首先是Snell定律，也是初中物理学过的：

$$
\eta_i\sin\theta_i = \eta_t\sin\theta_t
$$

据此可以根据入射角$\theta_i$计算出折射角$\theta_t$。当按此式计算出的$\sin\theta_t \ge 1$时，就发生了全反射，使得折射量为零。

现在来计算入射辐射量和折射辐射量之间的关系。根据Fresnel公式，折射的能量在入射能量中占比满足：

$$
L_t\cos\theta_tdAd\omega_t = (1 - F_r)\cos\theta_idAd\omega_i
$$

对Snell定律两边求导易得：

$$
\frac{\eta_i}{\eta_t} = \frac{\cos\theta_t d\theta_t}{\cos\theta_i d\theta_i}
$$

代入上式可得：

$$
\frac{L_t}{L_i} = (1 - F_r)\frac{\eta_t^2}{\eta_i^2}
$$

这样一来，就能给出完整的BTDF了：

$$
f_t(\Phi \to x \to \Theta) = (1 - F_r)\frac{\eta_t^2\delta(\Theta - \mathrm{Trans}_x(\Phi))}{\eta_i^2\cos\langle N_x, \Phi\rangle}~~~~\Phi \in \mathcal H^2, \Theta \in -\mathcal H^2
$$

其中$\mathrm{Trans}_x(\Phi)$是按照Snell定律计算出的理想折射方向，易证明：

$$
\mathrm{Trans}_x(\Phi) = \left[\frac{\eta_i}{\eta_t}(\Phi\cdot N_x)- \cos\theta_t\right]N_x - \frac{\eta_i}{\eta_t}\Phi
$$

当然，发生全反射时，$f_t$的值强制归零。

我想不出什么东西会只有折射没有反射，那样也太违和了，所以完美折射的例子就用玻璃球吧，即反射和折射的结合体。该材质每次采样时以一半的概率选择反射，以另一半的概率选择折射，并将计算结果乘二以达到无偏，效果如下方左侧所示：

![PathTracerExWithSpecularSampling]({{site.url}}/postpics/Atrc/2018_10_24_glass.png)

这噪点不免也太多了点，得改进一下。注意到当入射光线和法线方向相近时，Fresnel项几乎是零，折射成分居多；而当入射光线和法线接近垂直时，Fresnel项几乎是一，反射成分居多。据此，在进行折射/反射的选择时，可以先计算出Fresnel项$F_r$来，然后以$F_r$的概率选择反射，这是重要性采样的思想。应用这一策略后的效果如上图右侧所示，和左边相比改善颇多。

需要注意的是，玻璃材质的反射项计算也需要判断是否发生了全反射，这一点是上面的Perfect Specular Reflection中没有考虑的。反射和折射是相当难以调试的材质，因为即使代码中有问题，它往往也能呈现出看起来很不错的效果。

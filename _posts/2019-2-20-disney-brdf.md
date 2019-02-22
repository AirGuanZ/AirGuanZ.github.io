---
layout: post
title: Disney Principled BRDF：原理与实现
key: t20190220
tags:
  - Atrc
  - Graphics
---

Physically Based Rendering（PBR）是个很美好的概念，意为在物理意义上有据可依的渲染技术。PBR在物体材质上的应用主要体现在各种材质模型上，旨在建立统一规范的材质workflow，以及减少美术工作者在调参上花费的时间。然而，诸如微表面法线分布函数、导体的复数折射率等花里胡哨的公式和概念对使用者极不友好。此时，为PBR材质提供一组直观的参数和编辑方式，就成了推广该项技术的当务之急。Disney Principled BRDF（以后简称Disney BRDF）就是这样一项成果，在“直观”、“多样”和“基于物理”三者间取得了很好的均衡。本文将介绍其原理和实现方法。

<!--more-->

本文大部分内容基于[Disney BRDF Shader](https://github.com/wdas/brdf/blob/master/src/brdfs/disney.brdf)和[Disney Principled BRDF文档](https://disney-animation.s3.amazonaws.com/library/s2012_pbs_disney_brdf_notes_v2.pdf)。

## 参数概览

![PICTURE]({{site.url}}/postpics/disney-brdf-parameters.png)

如上图所示，Disney BRDF有11项用于调节材质外观的参数，它们分别是：

1. baseColor，材质的基本颜色，通常设定为常数或由贴图提供。
2. subsurface，用于控制材质的漫反射成分向次表面散射靠拢的程度。
3. metallic，金属度，指材质的外观向金属靠拢的程度。
4. specular，高光度，用于替代PBR中常见的折射率参数。
5. specularTint，高光颜色向基本颜色靠拢的程度。
6. roughness，材质粗糙程度。
6. anisotropic，各向异性度，即材质反射的非对称程度。
7. sheen，布料、纺织物材质的分量大小。
8. sheenTint，sheen分量的颜色向基本颜色靠拢的程度。
9. clearCoat，一个额外的高光项，用于模拟清漆的效果。
10. clearGloss，清漆的粗糙度。

以上所有参数的有效取值范围均为[0, 1]，该范围内的任何取值组合都被认为是合法的（valid）材质。下面依次解析Disney BRDF的各分量模型以及上述参数在其中起到的作用。

## 漫反射

中学物理课本告诉我们，漫反射是由于物体表面粗糙不平，会把入射光反射到各个不同方向上造成的反射现象。然而在PBR中，这正是微表面理论所建模的对象。微表面模型通常用于模拟高光，和漫反射的实际效果八竿子打不着，可见中学物理中的漫反射和PBR中的漫反射不是同一个概念。

[“漫反射”](https://en.wikipedia.org/wiki/Diffuse_reflection)可以认为是光进入材质表面以下发生浅层散射后再射出表面后的结果，在物理意义上和次表面散射是相同的。正因如此，许多材质模型会用fresnel公式计算折射光比例作为漫反射分量的乘积因子：

$$
(1 - F_r(\theta_i))(1 - F_r(\theta_o))
$$

乘积中的两项分别对应光进入材质中和离开材质时的折射比例。不过，Disney BRDF使用了魔改的fresnel公式——他们使用Schlick公式来作为fresnel项的近似，并且丢弃了折射率的概念，转而让fresnel项和物体表面的粗糙度挂钩。我没看出这有什么道理，不过原文称“这能很好地拟合实际数据，对artists也很友好”，那就暂且接受吧。公式如下：

$$
\begin{aligned}
&f_\mathrm{diffuse} = \frac{\mathrm{baseColor}}{\pi}(1 + (F_{D90} - 1)(1 - \cos\theta_i)^5)(1 + (F_{D90} - 1)(1 - \cos\theta_o)^5) \\
&F_{D90} = 0.5 + 2\cos^2\theta_d\mathrm{roughness}
\end{aligned}
$$

其中$\theta_d$是入射方向$\boldsymbol w_i$与half vector $\boldsymbol \omega_h = \mathrm{normalize(\boldsymbol w_i + \boldsymbol w_o)}$的夹角。

既然漫反射和次表面散射具有类似的原理，那么就可以用一个参数来在二者之间进行过度，也就是Disney BRDF中的subsurface参数。Disney BRDF会计算出一个漫反射值和一个次表面散射值，然后用subsurface在二者间进行插值。

对于次表面散射的计算，Disney BRDF并未使用人们所熟知的Jensen BSSRDF等复杂模型（否则计算效率也太低了），而是用一个BRDF来近似计算。他们的次表面散射公式是从Hanrahan-Krueger BRDF Approximation Of Isotropic BSSRDF改进而来的：

$$
\begin{aligned}
&f_\mathrm{subsurface} = 1.25\frac{\mathrm{baseColor}}{\pi}(F_{ss} (1 / (\cos\theta_i + \cos\theta_o) - 0.5) + 0.5) \\
&F_{ss} = (1 + (F_{ss90} - 1)(1 - \cos\theta_i)^5)(1 + (F_{ss90} - 1)(1 - \cos\theta_o)^5) \\
&F_{ss90} = \cos^2\theta_d\mathrm{roughness}
\end{aligned}
$$

## 高光项

如今几乎所有的PBR材质模型中的高光都是用Torrance-Sparrow微表面模型来模拟的，其形式如下：

$$
f = F_r(\theta_d)\frac{D(\theta_h)G(\theta_i, \theta_o)}{4\cos\theta_i\cos\theta_o}
$$

其中$F_r(\theta_d)$是fresnel项；$D(\theta_h, \phi_h)$是微表面法线分布函数值，$\theta_h$为$\boldsymbol w_h$与法线的夹角，$\phi_h$为$\boldsymbol w_h$的水平极角；$G(\theta_i, \theta_o)$为法线为$\boldsymbol w_h$的微表面中没有被其他微表面遮蔽的比例。$D$和$G$在不同的材质模型中有不同的选择，下方的$4\cos\theta_i\cos\theta_o$则是Torrance-Sparrow模型固有的一部分。Torrance-Sparrow模型的来历可简单参见[这里]({{site.url}}/2018/10/22/reflection-models.html#torrance-sparrow-model)。

### 微表面法线分布

Disney BRDF中的高光项也使用了该模型，其微表面法线分布函数具有如下形式：

$$
D(\theta_h, \phi_h) = \frac{c}{\left(\sin^2\theta_h\left(\dfrac{\cos^2\phi}{\alpha_x^2} + \dfrac{\sin^2\phi}{\alpha_y^2}\right) + \cos^2\theta_h\right)^\gamma}
$$

其中c是归一化常数；$\alpha_x$和$\alpha_y$是衡量表面粗糙度的参数，它们相等时材料为各向同性，不等时则为各向异性；$\gamma$是一个用来调整函数曲线的参数，为2时就恰好等价于近来流行的GGX（Trowbridge-Reitz）函数。由于这一函数在Trowbridge-Reitz的基础上添加了$\gamma$参数，因此被称为Generalized-Trowbridge-Reitz函数，简称GTR。Disney BRDF中有两处使用了GTR函数，一处是这里的高光项，另一处是清漆的反射。在高光项中，$\gamma$值恰好被设定为2。$\alpha_x$、$\alpha_y$与Disney BRDF参数间的关系为：

$$
\begin{aligned}
\alpha_x &= \mathrm{roughness}^2 a \\
\alpha_y &= \mathrm{roughness}^2 / a \\
a &= \sqrt{1 - 0.9\mathrm{anisotropic}}
\end{aligned}
$$

考虑到$D$应满足归一化约束：

$$
\int_{\mathcal H^2}\cos\theta_h D(\theta_h, \phi_h)d\omega_h = 1
$$

取$\gamma = 2$，稍微积个分：

$$
\begin{aligned}
&\int_{\mathcal H^2}\frac{c\cos\theta_h}{\left(\sin^2\theta_h\left(\frac{\cos^2\phi}{\alpha_x^2} + \frac{\sin^2\phi}{\alpha_y^2}\right) + \cos^2\theta\right)^2}d\omega_h \\
= &c\int_{0}^{2\pi}\int_{0}^{\pi/2}\frac{\sin\theta_h\cos\theta_h}{\left(\sin^2\theta_h\left(\frac{\cos^2\phi}{\alpha_x^2} + \frac{\sin^2\phi}{\alpha_y^2}\right) + \cos^2\theta\right)^2}d\theta_h d\phi_h \\
= &\frac c 2 \int_0^{2\pi}\frac 1 {\frac{\cos^2\phi_h}{\alpha_x^2} + \frac{\sin^2\phi_h}{\alpha_y^2}}d\phi_h
= \pi\alpha_x\alpha_yc = 1
\end{aligned}
$$

解得$c = 1 / (\pi\alpha_x\alpha_y)$，即：

$$
D(\theta_h, \phi_h) = \frac 1 {\pi\alpha_x\alpha_y\left(\sin^2\theta_h\left(\frac{\cos^2\phi}{\alpha_x^2} + \frac{\sin^2\phi}{\alpha_y^2}\right) + \cos^2\theta_h\right)^2}
$$

这公式写着太费劲了，不妨设：

$$
\Phi(\phi_h) = \frac{\cos^2\phi_h}{\alpha_x^2} + \frac{\sin^2\phi_h}{\alpha_y^2}
$$

于是：

$$
D(\theta_h, \phi_h) = \frac 1 {\pi\alpha_x\alpha_y\left(\sin^2\theta_h\Phi(\phi_h) + \cos^2\theta_h\right)^2}
$$

如果要在离线渲染中使用这一分布函数，我们还需要对BRDF进行重要性采样。具体到这里，我们选择按$D(\theta_h, \phi_h)\cos\theta_h$设计采样所用的概率密度函数。令$p_h(\theta_h, \phi_h)d\theta_hd\phi_h = D(\theta_h, \phi_h)\cos\theta_hd\omega_h$，那么：

$$
\begin{aligned}
&p_h(\theta_h, \phi_h)d\theta_hd\phi_h = D(\theta_h, \phi_h)\cos\theta_hd\omega_h = D(\theta_h, \phi_h)\sin\theta_h\cos\theta_hd\theta_hd\phi_h \\
\Rightarrow~&p_h(\theta_h, \phi_h) = D(\theta_h, \phi_h)\sin\theta_h\cos\theta_h
\end{aligned}
$$

两边在$[0, \pi/2]$上对$\theta_h$积分，得：

$$
\int_0^{\pi/2}p_h(\theta_h, \phi_h)d\theta_h = \int_0^{\pi/2}D(\theta_h, \phi_h)\sin\theta_h\cos\theta_hd\theta_h~\Rightarrow~p_h(\phi_h) = \frac 1 {2\pi\alpha_x\alpha_y\Phi(\phi)}
$$

自然而然地：

$$
p_h(\theta_h\mid\phi_h) = \frac{p_h(\theta_h, \phi_h)}{p_h(\phi_h)} = 2\pi\alpha_x\alpha_y\Phi(\phi)D(\theta_h, \phi_h)\sin\theta_h\cos\theta_h
$$

有了概率密度函数$p_h(\phi_h)$和$p_h(\theta_h\mid\phi_h)$，对其积分就能得到分布函数$P_h(\phi_h)$和$P_h(\theta_h\mid\phi_h)$。首先是$P_h(\phi_h)$：

$$
P_h(\phi_h) = \int_0^{\phi_h}p_h(x)dx = \frac 1 {2\pi\alpha_x\alpha_y}\int_0^{\phi_h}\frac 1 {\Phi(x)}dx = \frac 1 {2\pi}\arctan\left(\frac{\alpha_x}{\alpha_y}\tan \phi\right)
$$

设$\xi_1$是$[0, 1)$上的服从均匀分布的随机变量，令$P_h(\phi_h) = \xi_1$，解得：

$$
\phi_h = \arctan\left(\frac{\alpha_y}{\alpha_x}\tan(2\pi\xi_1)\right)
$$

这个式子有些问题：里面的$\tan$对$\xi_1 = 0.25$或$\xi_1 = 0.75$是无意义的，外面的$\arctan$的取值范围又是$(-\pi/2, \pi/2)$而不是我们想要的$[0, \pi/2)$，如下图所示：

![PICTURE]({{site.url}}/postpics/sample-phi-in-aniso-ggx.png)

可以看到这一函数被分为三段，我们把中间的一段向上平移$\pi$，最后一段向上平移$2\pi$，并专门处理$0.25$和$0.75$附近的值以保持函数曲线平滑、避免出现数值问题即可。事实上，我们并不直接需要$\phi_h$的值，只要求出$\sin\phi_h$和$\cos\phi_h$即可，这样就能绕开$\tan$和$\arctan$的糟糕性质——

$$
\begin{aligned}
\sin\phi_h &= \frac{\alpha_y}{r}\sin(2\pi\xi_1) \\
\cos\phi_h &= \frac{\alpha_x}{r}\cos(2\pi\xi_1)
\end{aligned}
$$

其中$r$是保证$\sin^2\phi_h + \cos^2\phi_h = 1$的归一化系数。

最后是求$p_h(\theta_h\mid\phi_h)$对应的条件分布：

$$
\begin{aligned}
P_h(\theta_h\mid\phi_h) &= \int_0^{\theta_h}2\pi\alpha_x\alpha_y\Phi(\phi)D(x, \phi_h)\sin x\cos xdx \\
&= \int_0^{\theta_h}\frac {2\Phi(\phi_h)\sin x\cos x} {\left(\sin^2x\Phi(\phi_h) + \cos^2x\right)^2}dx \\
&= \frac{\Phi(\phi_h)(1 - \cos(2\theta_h))}{(1 - \Phi(\theta_h))\cos(2\theta_h) + (1 + \Phi(\theta_h))}
\end{aligned}
$$

令$P_h(\theta_h\mid\phi_h) = \xi_2$，解得：

$$
\begin{aligned}
&\frac{\Phi(\phi_h)(1 - \cos(2\theta_h))}{(1 - \Phi(\theta_h))\cos(2\theta_h) + (1 + \Phi(\theta_h))} = \xi_2 \\
\Rightarrow~&\theta_h = \arccos\left(\sqrt{\frac{\Phi(\phi_h)(1 - \xi_2)}{(1 - \Phi(\phi_h))\xi_2 + \Phi(\phi_h)}}\right)
\end{aligned}
$$

最后推导一下按这种方式采样$\boldsymbol \omega_h$，然后依据它计算出入射方向$\boldsymbol \omega_i$的概率密度。注意到：

$$
p(\boldsymbol \omega_h)d\omega_h = p(\theta_h, \phi_h)d\theta_hd\phi_h \Rightarrow p(\boldsymbol \omega_h) = D(\theta_h, \phi_h)\cos\theta_h
$$

又根据$d\omega_i = 4\cos\theta_hd\omega_h$，有：

$$
p(\boldsymbol \omega_i) = p(\boldsymbol \omega_h)\frac{d\omega_h}{d\omega_i} = \frac{p(\boldsymbol \omega_h)}{4\cos\theta_h} = \frac{D(\theta_h, \phi_h)}{4}
$$

### 遮蔽项

接下来是Torrance-Sparrow公式中的$G$，Disney BRDF选择了Smith遮蔽函数：

$$
G(\boldsymbol \omega_i, \boldsymbol \omega_o) = G_1(\boldsymbol \omega_i)G_1(\boldsymbol \omega_o)
$$

至于$G_1$，就直接用各向异性GGX分布对应的遮蔽函数好了：

$$
\begin{aligned}
&G_1(\boldsymbol \omega) = \frac 1 {1 + \Lambda(\boldsymbol \omega)} \\
&\Lambda(\boldsymbol \omega) = -\frac 1 2 + \frac 1 2 \sqrt{1 + (\alpha_x^2\cos^2\phi + \alpha_y^2\sin^2\phi)\tan^2\theta}
\end{aligned}
$$

注意到我们对高光进行采样的时候没有考虑$G$的影响，其主要原因是，emmm，考虑了$G$之后我就推不出来了……

（施工中……）

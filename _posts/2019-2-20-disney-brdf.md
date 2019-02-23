---
layout: post
title: Disney Principled BRDF实现笔记
key: t20190220
tags:
  - Atrc
  - Graphics
---

Physically Based Rendering（PBR）是个很美好的概念，意为在物理意义上有据可依的渲染技术。PBR在物体材质上的应用主要立足于各种反射/折射模型上，然而，诸如微表面法线分布函数、导体的复数折射率等花里胡哨的公式和概念对使用者极不友好。Disney Principled BRDF（以后简称Disney BRDF）为PBR材质提供一组直观的参数和编辑方式，在“直观”、“多样”和“基于物理”三者间取得了很好的均衡。

<!--more-->

本文记叙了我在实现Disney BRDF过程中的推导和所使用的公式，而不是解释一些PBR相关的基础知识。本文大部分内容以[Disney BRDF Shader](https://github.com/wdas/brdf/blob/master/src/brdfs/disney.brdf)和[Disney Principled BRDF文档](https://disney-animation.s3.amazonaws.com/library/s2012_pbs_disney_brdf_notes_v2.pdf)为参考。

初入图形学，如有错误，欢迎指正。

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
10. clearGloss，清漆的光滑程度。

以上所有参数的有效取值范围均为[0, 1]，该范围内的任何取值组合都被认为是合法的（valid）材质。下面依次解析Disney BRDF的各分量模型以及上述参数在其中起到的作用。

## 漫反射

[漫反射](https://en.wikipedia.org/wiki/Diffuse_reflection)是光进入材质表面以下发生浅层散射后再从离入射点非常近的位置射出的结果，在物理意义上和次表面散射是相同的（只是尺度不同）。正因如此，许多材质模型会用fresnel公式计算折射光比例作为漫反射分量的乘积因子。

不过，Disney BRDF使用了魔改的fresnel公式——他们使用Schlick公式来作为fresnel项的近似，并且丢弃了折射率的概念，转而让fresnel项和物体表面的粗糙度挂钩。我没看出这有什么道理，不过原文称“这能很好地拟合实际数据，对artists也很友好”，那就暂且接受吧。公式如下：

$$
\begin{aligned}
&f_\mathrm{diffuse} = \frac{\mathrm{baseColor}}{\pi}(1 + (F_{D90} - 1)(1 - \cos\theta_i)^5)(1 + (F_{D90} - 1)(1 - \cos\theta_o)^5) \\
&F_{D90} = 0.5 + 2\cos^2\theta_d\mathrm{roughness}
\end{aligned}
$$

其中$\theta_d$是入射方向$\boldsymbol w_i$与half vector $\boldsymbol \omega_h = \mathrm{normalize(\boldsymbol w_i + \boldsymbol w_o)}$的夹角。

既然漫反射和次表面散射具有类似的原理，那么就可以用一个参数来在二者之间过渡，也就是Disney BRDF中的subsurface参数。Disney BRDF会计算出一个漫反射值和一个次表面散射值，然后用subsurface在二者间进行插值。

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

其中$c$是归一化常数；$\alpha_x$和$\alpha_y$是衡量表面粗糙度的参数，它们相等时材料为各向同性，不等时则为各向异性；$\gamma$是一个用来调整函数曲线的参数，为2时就恰好等价于近来流行的GGX（Trowbridge-Reitz）函数。由于这一函数在Trowbridge-Reitz的基础上添加了$\gamma$参数，因此被称为Generalized-Trowbridge-Reitz函数，简称GTR。Disney BRDF中有两处使用了GTR函数，一处是这里的高光项，另一处是清漆的反射。在高光项中，$\gamma$值恰好被设定为2。$\alpha_x$、$\alpha_y$与Disney BRDF参数间的关系为：

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
\int_0^{\pi/2}p_h(\theta_h, \phi_h)d\theta_h = \int_0^{\pi/2}D(\theta_h, \phi_h)\sin\theta_h\cos\theta_hd\theta_h~\Rightarrow~p_h(\phi_h) = \frac 1 {2\pi\alpha_x\alpha_y\Phi(\phi_h)}
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

我在对高光进行采样的时候没有考虑fresnel项和遮蔽项，主要原因是考虑了之后我就推不出来了，emmmm……

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

其中的$\alpha_x$和$\alpha_y$还是按之前的方法通过roughness和anisotropic计算出来的。值得一提的是，最初的Disney BRDF在计算高光的遮蔽项时将roughness做了如下变换：

$$
\mathrm{roughness}_g = \frac{1 + \mathrm{roughness}}{2}
$$

并称这样能够更好地拟合MERL数据库中的材质数据。后来随着理论分析的进步，他们认为MERL中对光滑材料的数据测量并不准确，去掉了这一trick。

## 边缘处的光泽

当我们从几乎垂直于法线的方向去观察诸如丝绸或一些布料时，它们会显得比普通的漫反射更明亮一些。Disney BRDF直接用一个缩放后的Schlick公式去模拟这一现象，即$(1 - \cos\theta_d)^5\mathrm{sheen}$。

## 清漆项

诸如车漆、木质地板等材料可以通过一个两层模型来渲染：上面是一层透明材料，通常比较光滑，下面则是漫反射或金属等其他材料。如果要仔细地渲染这一模型，我们需要考虑两层间的多次反射和折射，并用fresnel公式来分配每次反射/折射的比例。Disney BRDF采用的方案要简单粗暴得多——加上一个额外的高光，称为清漆（clearcoat）项。

清漆项同样采用Torrance-Sparrow模型，其遮蔽项是固定取粗糙度为0.25的GGX遮蔽项，fresnel项固定取折射率为1.5的绝缘体，微表面法线分布函数则是取$\gamma = 1$的各向同性GTR函数（记作GTR1）。下面稍微推以下GTR1的归一化系数和采样方法。

### GTR1函数

GTR1具有如下形式：

$$
D(\theta_h) = \frac c {\alpha^2\cos^2\theta_h + \sin^2\theta_h}
$$

其中$c$是归一化系数。利用归一化约束：

$$
\int_{\mathcal H^2}D(\theta_h)\cos\theta_hd\omega_h = c\int_0^{2\pi}\int_0^{\pi/2}\frac{\sin\theta_h\cos\theta_h}{\alpha^2\cos^2\theta_h + \sin^2\theta_h}d\theta_hd\phi_h = \frac{2\pi c}{\alpha^2 - 1}\ln\alpha = 1
$$

解得$c = (\alpha^2 - 1)/(2\pi\ln\alpha)$，于是：

$$
D(\theta_h) = \frac {\alpha^2 - 1} {2\pi\ln\alpha(\alpha^2\cos^2\theta_h + \sin^2\theta_h)}
$$

接下来是重要性采样。由于GTR1对$\phi_h$而言是各向同性的，我们可以直接取$\phi_h = 2\pi\xi_1$，并且$\theta_h$和$\phi_h$相互独立，于是可以求出$p_h(\theta_h)$：

$$
p_h(\theta_h) = \frac{p_h(\theta_h, \phi_h)}{p_h(\phi_h)} = 2\pi D(\theta_h)\cos\theta_h\sin\theta_h = \frac {(\alpha^2 - 1)\cos\theta_h\sin\theta_h} {(\alpha^2\cos^2\theta_h + \sin^2\theta_h)\ln\alpha}
$$

积分之：

$$
\begin{aligned}
&P_h(\theta_h) = \int_0^{\theta_h}p_h(t)dt = \int_0^{\theta_h}\frac {(\alpha^2 - 1)\cos t\sin t} {(\alpha^2\cos^2 t + \sin^2 t)\ln\alpha}dt \\
= &\frac 1 {2\ln\alpha}\ln\left[\frac{2\alpha^2}{(\alpha^2 - 1)\cos(2\theta_h) + (\alpha^2 + 1)}\right]
\end{aligned}
$$

令$P_h(\theta_h) = \xi_2$，解得：

$$
\theta_h = \arccos\left(\sqrt{\frac{\alpha^{2 - 2\xi_2} - 1}{\alpha^2 - 1}}\right)
$$

## BRDF

上面说了这么多“项”，我们还需要将它们融合起来变成一个BRDF。能量守恒这种高端要求是不指望了（像Smith G这种函数本身就会带来能量损失），但至少看起来不能太离谱。设：

$$
\begin{aligned}
C &= \mathrm{baseColor} \\
\sigma_m &= \mathrm{metallic} \\
\sigma_{ss} &= \mathrm{subsurface} \\
\sigma_s &= \mathrm{specular} \\
\sigma_{st} &= \mathrm{specularTint} \\
\sigma_r &= \mathrm{roughness} \\
\sigma_a &= \mathrm{anisotropic} \\
\sigma_{sh} &= \mathrm{sheen} \\
\sigma_{sht} &= \mathrm{sheenTint} \\
\sigma_{c} &= \mathrm{clearcoat} \\
\sigma_{cg} &= \mathrm{clearcoatGloss}
\end{aligned}
$$

并设入射方向$\boldsymbol \omega_i$为$(\theta_i, \phi_i)$，出射方向$\boldsymbol \omega_o$为$(\theta_o, \phi_o)$，half vector $\boldsymbol \omega_h$为$(\theta_h, \phi_h)$，$\boldsymbol \omega_i$与$\boldsymbol \omega_h$的夹角为$\theta_d$，则Disney BRDF为：

$$
\begin{aligned}
f_\text{disney}(\boldsymbol \omega_i, \boldsymbol \omega_o) =~&(1 - \sigma_m)\left(\frac{C}{\pi}\mathrm{mix}(f_d(\boldsymbol \omega_i, \boldsymbol \omega_o), f_{ss}(\boldsymbol \omega_i, \boldsymbol \omega_o), \sigma_{ss}) + f_{sh}(\boldsymbol \omega_i, \boldsymbol \omega_o)\right) \\
+~&\frac{F_s(\theta_d)G_s(\boldsymbol \omega_i, \boldsymbol \omega_o)D_s(\boldsymbol \omega_h)}{4\cos\theta_i\cos\theta_o} \\
+~&\frac {\sigma_c} 4 \frac{F_c(\theta_d)G_c(\boldsymbol \omega_i, \boldsymbol \omega_o)D_c(\boldsymbol \omega_i, \boldsymbol \omega_o)}{4\cos\theta_i\cos\theta_o}
\end{aligned}
$$

其中$f_d$为漫反射：

$$
\begin{aligned}
f_d(\boldsymbol \omega_i, \boldsymbol \omega_o) &= (1 + (F_{D90} - 1)(1 - \cos\theta_i)^5)(1 + (F_{D90} - 1)(1 - \cos\theta_o)^5) \\
F_{D90} &= 0.5 + 2\cos^2\theta_d\sigma_r
\end{aligned}
$$

$f_{ss}$为次表面散射项：

$$
\begin{aligned}
f_{ss}(\boldsymbol \omega_i, \boldsymbol \omega_o) &= 1.25(F_{ss} (1 / (\cos\theta_i + \cos\theta_o) - 0.5) + 0.5) \\
F_{ss} &= (1 + (F_{ss90} - 1)(1 - \cos\theta_i)^5)(1 + (F_{ss90} - 1)(1 - \cos\theta_o)^5) \\
F_{ss90} &= \cos^2\theta_d\sigma_r
\end{aligned}
$$

$f_{sh}$为sheen项：

$$
\begin{aligned}
f_{sh}(\boldsymbol \omega_i, \boldsymbol \omega_o) &= \mathrm{mix}(\mathrm{one}, C_{tint}, \sigma_{sht})\sigma_{sh}(1 - \cos\theta_d)^5 \\
C_{tint} &= \frac{C}{\mathrm{lum}(C)}
\end{aligned}
$$

$\mathrm{lum}$是求[相对亮度](https://en.wikipedia.org/wiki/Relative_luminance)的函数，对线性颜色空间中的RGB值$C$，有：

$$
\mathrm{lum}(C) \approx 0.2126 \times C.r + 0.7152 \times C.g + 0.0722 \times C.b
$$

$F_s$是高光的frensel项，用Schlick公式做了近似。对绝缘体而言，Schlick公式中的$R_0$可以用一个实数表示；而对金属而言，$R_0$是带有颜色信息的，因此Disney BRDF利用金属度参数在两种情况间进行了插值：

$$
\begin{aligned}
F_s(\theta_d) &= C_s + (1 - C_s)(1 - \cos\theta_d)^5 \\
C_s &= \mathrm{mix}(0.08\sigma_s\mathrm{mix}(\mathrm{one}, C_{tint}, \sigma_{st}), C, \sigma_m)
\end{aligned}
$$

$G_s$是高光的遮蔽项，采用各向异性GGX对应的Smith函数：

$$
\begin{aligned}
G_s(\boldsymbol \omega_i, \boldsymbol \omega_o) &= G_{s1}(\boldsymbol \omega_i)G_{s1}(\boldsymbol \omega_o) \\
G_{s1}(\boldsymbol \omega) &= \frac 1 {1 + \Lambda_s(\boldsymbol \omega)} \\
\Lambda_s(\boldsymbol \omega) &= -\frac 1 2 + \frac 1 2 \sqrt{1 + (\alpha_x^2\cos^2\phi + \alpha_y^2\sin^2\phi)\tan^2\theta}
\end{aligned}
$$

$D_s$是高光的微表面法线分布函数，采用各向异性GTR2函数（其实就是GGX）：

$$
\begin{aligned}
D_s(\boldsymbol \omega_h) &= \frac 1 {\pi\alpha_x\alpha_y\left(\sin^2\theta_h\left(\frac{\cos^2\phi}{\alpha_x^2} + \frac{\sin^2\phi}{\alpha_y^2}\right) + \cos^2\theta_h\right)^2} \\
\alpha_x &= \sigma_r^2 a \\
\alpha_y &= \sigma_r^2 / a \\
a &= \sqrt{1 - 0.9\sigma_a}
\end{aligned}
$$

$F_c$是清漆的fresnel项，采用折射率固定为1.5的绝缘体对应的Schlick公式：

$$
F_c(\theta_d) = 0.04 + 0.96(1 - \cos\theta_d)^5
$$

$G_c$是清漆的遮蔽项，采用粗糙度为0.25的各向同性GGX对应的Smith函数：

$$
\begin{aligned}
G_c(\boldsymbol \omega_i, \boldsymbol \omega_o) &= G_{c1}(\boldsymbol \omega_i)G_{c1}(\boldsymbol \omega_o) \\
G_{c1}(\boldsymbol \omega) &= \frac 1 {1 + \Lambda_c(\boldsymbol \omega)} \\
\Lambda_c(\boldsymbol \omega) &= -\frac 1 2 + \frac 1 2 \sqrt{1 + \alpha^2\tan^2\theta}
\end{aligned}
$$

$G_c$居然没和$\sigma_{cg}$挂钩，官方解释是这样效果看起来很好，太真实了……

$D_c$是清漆的微表面法线分布，采用各向同性的GTR1函数：

$$
\begin{aligned}
D_c(\boldsymbol \omega_h) &= \frac {\alpha^2 - 1} {2\pi\ln\alpha(\alpha^2\cos^2\theta_h + \sin^2\theta_h)} \\
\alpha &= \mathrm{mix}(0.1, 0.01, \sigma_{cg})
\end{aligned}
$$

至此，Disney BRDF的计算就介绍完毕，可以在Shader中实现了。

## 重要性采样

之前我们讨论了两个高光项的重要性采样，然鹅这并不是针对整个Disney BRDF的，因此需要将这些采样技术统合到一个采样方案中。现假设给定了$\boldsymbol \omega_o$，需要采样入射光的方向$\boldsymbol \omega_i$。

Sheen在整个BRDF中占比太小，所以不参与采样。我们主要考虑的采样对象有三个：漫反射，高光和清漆。我们简单地以正比于强度的离散分布律来选择采样哪一种反射，然后用[多重重要性采样]({{site.url}}/2018/10/15/multiple-importance-sampling.html)技术将其概率密度函数值结合起来。

令baseColor为白色（具体数值不重要，重要的是各颜色通道相同），保持其他参数不变，此时的$(1 - \sigma_m)$，$F_s(\theta_d)$和$\sigma_cF_c(\theta_d)/4$将被作为三种采样对象的权重。设这三个权重在归一化后分别为$w_d, w_s, w_c$，$p_d, p_s, p_c$分别为漫反射、高光和清漆在采样$\boldsymbol \omega_i$时的概率密度函数，那么总的概率密度函数为：

$$
p(\boldsymbol \omega_i) = w_dp_d(\boldsymbol \omega_i) + w_sp_s(\boldsymbol \omega_i) + w_cp_c(\boldsymbol \omega_i)
$$

对漫反射，我们可以对法线所在的半球立体角使用cosine-weighted采样；对高光和清漆，可以分别使用之前推导的GTR2采样与GTR1采样技术。这样一来，就得到了对Disney BRDF进行采样的方法，可以将其实现在离线渲染中。

## 实现效果

（施工中……）

## NEXT STEP

Disney BRDF的一个显著缺点是无法处理透明和半透明的材质，这在之后的扩展版——Disney Principled BSDF中得到了解决。BSDF版本不仅引入了透射，还支持了介质吸收和Normalized Diffusion BSSRDF，这需要渲染算法加以支持。不出意外的话，我以后会有一篇关于它的实现笔记。

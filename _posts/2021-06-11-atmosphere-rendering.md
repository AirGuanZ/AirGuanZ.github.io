---
title: 预计算大气散射模型：原理与实现
key: t20210611
tags:
  - Graphics
---

用自己的语言整理一下最近阅读的实时大气渲染方法，主要参考《A Scalable and Production Ready Sky and Atmosphere Rendering Technique》和《Precomputed Atmospheric Scattering》两文。

<!--more-->

## 大气散射模型

我们把大气散射模型建立在对“星球”这一概念的近似上。星球大致是一个半径为$R$的球体，在此基础上通过一个高度场来描述地表起伏。在地表以外、半径小于$R'$（显然$R < R'$）的空间中存在着非均匀的大气层。由于大气中的空气折射率随着位置的不同而不同，光线穿过大气层时会弯曲，这在实时渲染中很难模拟，因而本文忽略光线的偏转现象。

### 渲染方程

对介质中或物体表面的某个位置$x$，从$x$向方向$\omega$传输的辐射亮度（radiance）$L(x \to \omega)$由下述渲染方程给出：

$$
\begin{aligned}
L(x \to \omega)

&= L_e(x \to \omega) \\

&+~ T(x, y)\int_{\mathcal S^2}f_s(\omega_i \to y \to -\omega)|\cos\langle n_y, \omega_i\rangle|L(y \leftarrow \omega_i)d\omega_i \\

&+~ \int_x^yT(x, p)\sigma_s(p)\left(\int_\mathcal{S^2}\rho(\omega_i \to p \to -\omega)L(\omega_i \to p)d\omega_i\right)dl_p
\end{aligned}
$$

其中$y = \mathrm{raycast}(x \to -\omega)$；$\rho$是介质的相函数（phase function）；$f_s$是物体表面的BSDF；$T(a, b)$表示$a, b$之间的透射率（transmittance），其定义为：

$$
T(a, b) = \exp\left(-\int_a^b \sigma_t(p)dl_p\right)
$$

$\sigma_s$为介质散射系数，$\sigma_t$为介质衰减系数。更多关于此渲染方程的相关内容可见文[介质渲染](https://airguanz.github.io/2018/10/28/LTE-with-participating-medium.html)，这里只是回顾一下符号，不再赘述。

### Rayleigh

在大气层中，空气分子带来的散射可以用[Rayleigh theory](https://en.wikipedia.org/wiki/Rayleigh_scattering)描述：

$$
\begin{aligned}
\sigma_{Rt}(h, \lambda) &= \sigma_{Rs}(h, \lambda) & \text{Rayleigh衰减率} \\
\sigma_{Rs}(h, \lambda) &= \sigma_{Rs0}\frac{8\pi^3(n^2-1)^2}{3N\lambda^4}e^{-\frac{h}{H_R}} & \text{Rayleigh散射率} \\
\rho_R(\mu) &= \frac{3}{16\pi}(1+\mu^2) & \text{Rayleigh散射相函数}
\end{aligned}
$$

其中，$h$是位置的海拔高度（与球心的距离减去球面半径$R$），$\lambda$是波长，$n$是空气折射率，$N$是海平面$R$处的空气分子密度，$\mu$表示$\cos\langle-\omega_i, \omega_o\rangle$，$H_R$是一个与大气性质相关的常数。

### Mie

除了Rayleigh散射外，大气中还存在小颗粒（气溶胶，小水珠等）带来的散射与吸收现象，由[Mie theory](https://en.wikipedia.org/wiki/Mie_scattering)描述：

$$
\begin{aligned}
\sigma_{Mt}(h, \lambda) &= \sigma_{Ms}(h, \lambda) + \sigma_{Ma}(h, \lambda) & \text{Mie衰减率} \\
\sigma_{Ms}(h, \lambda) &= \sigma_{Rs}(0, \lambda)e^{-\frac{h}{H_M}} & \text{Mie散射率} \\
\sigma_{Ma}(h, \lambda) &= \beta \sigma_{Ms}(h, \lambda) & \text{Mie吸收率} \\
\rho_M(\mu) &= \frac{3}{8\pi}\frac{(1-g^2)(1+\mu^2)}{(2+g^2)(1+g^2-2g\mu)^{3/2}} & \text{Mie散射相函数}
\end{aligned}
$$

其中$g$是相函数的非对称因子；$\beta, H_M$均为常数。

### Ozone

臭氧层对地球大气层外观也有较大的影响，当太阳靠近地平线时，是臭氧的吸收效应使得天空的大部分区域呈现为蓝色。其吸收率可以由下式近似表示：

$$
\sigma_{Oa}(h, \lambda) = \sigma_{O0}(\lambda)\max\{0, 1 - \frac{|h - 25km|}{15km}\}
$$

### 汇总

在实时渲染中，我们不可能逐波长地去计算光的散射，而是采用RGB三个分量对光谱上的能量分布进行了近似，故$\lambda$参数也只能进行取定为某个值，记作$\lambda_0$。由此，我们在计算大气散射时使用的$\sigma_t, \sigma_s$以及$\rho$为：

$$
\begin{aligned}
\sigma_t(h) &= \sigma_{Rs}(h) + \sigma_{Mt}(h) + \sigma_{Oa}(h) \\
\sigma_s(h) &= \sigma_{Rs}(h) + \sigma_{Ms}(h) \\
\rho(\mu)   &= \frac{\sigma_{Rs}(h)}{\sigma_s(h)}\rho_R(\mu) + \frac{\sigma_{Ms}(h)}{\sigma_s(h)}\rho_M(\mu)
\end{aligned}
$$

## 拆解和近似

### 透射率

透射率$T(a, b)$描述了一束光从$a$点传播到$b$点后，尚未被吸收或衰减的辐射亮度比例。$T(a, b)$仅与$a$和$b$的位置有关，和地形无关，因此，我们可以在假定星球上没有任何凸起（即所有地表到球心的距离都是$R$）的情况下讨论$T$的计算。

对大气层中的某一点$p_o$和某个方向$\omega$，我们总是可以通过刚体坐标变换将$p_o$点变换到球心$O$的正上方，且$\omega$位于$xOy$平面上。此时，设$p_e$是射线$p = p_o + t\omega (t \ge 0)$与大气层外侧边缘或地表的最近交点，则$T(p_o, p_e)$仅依赖于两个标量：$r = \|p_o\|$以及$\omega$在$xOy$平面上的角度$\theta$：

<p align="center">
<img width="50%" height="50%" src="{{site.url}}/postpics/par/00.png">
</p>

我们将这一信息记录在预计算的表格$\mathbb T(r, \theta)$中。对大气中的任意两点$a, b$，$T(a, b)$都可以根据这个表格计算出来：

$$
T(a, b) = T(a, c) / T(b, c) = \mathbb T(r_a, \theta_{a\omega_{ab}}) / \mathbb T(r_b, \theta_{b\omega_{ab}})
$$

<p align="center">
<img width="50%" height="50%" src="{{site.url}}/postpics/par/01.png">
</p>

### 零散射项

设想从大气中某一点$p$向$\omega$发射一条射线，正好可以命中太阳，那么$L(p \leftarrow \omega)$无疑包含了从太阳直接发出的光。这部分辐射亮度可以由下式给出：

$$
L_0(p \leftarrow \omega) = L_e\mathbb T(r_p, \theta_{p\omega})
$$

其中，$L_e$是从太阳表面出发的辐射亮度。$L_0$可以通过在天空中单独绘制一个表示太阳的“圆盘”来添加，因此和其他的部分几乎没有实现上的耦合。

### 单次散射项

所谓单次散射，是指光自光源出发以来仅经过了一次介质散射后就进入了观察者的眼睛。大气层的散射率不高，因此我们观察到的光的大部分能量都来自单次散射光线。记单次散射的辐射亮度为$L_s$，那么：

$$
L_s(p\leftarrow\omega)=\int_p^{p_e}T(p, q)\sigma_s(h_q)\left(\int_\mathcal S^2\rho(\cos\langle \omega_i, \omega\rangle)L_0(p\leftarrow \omega_i)V_\text{sun}(q, \omega_i)d\omega_i\right)dl_q
$$

其中$V_\text{sun}$表示$q$点$\omega_i$方向上太阳的可见性（取值为0或1）。如果我们做一个简化，假设太阳在这里可以被视为一个理想方向光，用狄拉克函数$\delta_\text{sun}$用于选出太阳相对于$q$点的方向$\omega_\text{sun}$，$L_e$被替换为$E_e\delta_\text{sun}$，那么上式可以简化为：

$$
\begin{aligned}
L_s(p\leftarrow\omega) &= \int_p^{p_e}T(p, q)\sigma_s(h_q)\rho(\cos\langle\omega_\text{sun}, \omega\rangle) E_e\mathbb T(r_q, \theta_{q\omega_\text{sun}})V_\text{sun}(q)dl_q
\end{aligned}
$$

在实现中，$V_\text{sun}$可以通过Shadow Mapping技术求出，外层积分则在ray marching过程中近似计算，$T(p, q)$也在ray marching时一并求出：

$$
L_s(p \leftarrow \omega) = \sum_q T(p, q)\sigma_s(h_q)\rho(\cos\langle \omega_{\text{sun}}, \omega\rangle)E_e\mathbb T(r_q, \theta_{q\omega_\text{sun}})V_\text{sun}(q)\Delta l_q
$$

### 单次反射项

单次反射项是指从太阳出发，仅在地表反射一次，然后就进入观察者眼中的光。和之前一样，我们假设太阳光是方向光。记单次反射项为$L_r(p \leftarrow \omega)$，从$p$点往$\omega$方向观察到的地面点为$x$，地表BSDF为$f_s$，那么：

$$
\begin{aligned}
L_r(p \leftarrow \omega) &= T(p, x)f_s(\omega_\text{sun} \to x \to -\omega)\cos\langle n_x, \omega_\text{sun}\rangle E_eV_\text{sun}(x)T(x, \mathrm{raycast}(x, \omega_\text{sun})) \\
&= T(p, x)f_s(\omega_\text{sun} \to x \to -\omega)\cos\langle n_x, \omega_\text{sun}\rangle E_eV_\text{sun}(x)\mathbb T(r_x, \theta_{x\omega_\text{sun}})
\end{aligned}
$$

### 多次散射项

多次散射项确实比较难算，我们先引入几个简化：

* 对使用path tracing算法产生的ground truth观察可知，当散射次数增加时，相函数的非对称性对结果的影响会迅速下降，表现得越来越趋近于各向同性的相函数（即$p_u = 1/(4\pi)$），据此，我们在计算二次及以上次数的大气散射时，均可以尝试把相函数替换为$p_u$。
* 多次散射时的光线路径很难用像Shadow Mapping那样的技术来快速判定散射点之间的可见性，再考虑到这些路径遍布整个大气层，通常不会被遮挡得很严重，不妨在计算多次散射时忽略可见性函数，即假设$V$的值总是1。
* 忽略地面反射在多次散射中的贡献。

对大气层中的某一点$p$，下式描述了$p$处恰好经过两次散射后的内散射（in-scattering）项：

$$
L_{2}(p) = \int_{\mathcal S^2}(L_s(p \leftarrow \omega) + L_r(p\leftarrow \omega))\rho_ud\omega
$$

对应地，三次散射项为：

$$
L_3(p) = \int_{\mathcal S^2}\left(\int_0^D T(p, q)\sigma_s(q)L_2(p)dl_q\right)\rho_ud\omega~~~\text{where}~~~D = |\mathrm{raycast}(p, \omega) - p|
$$

这里忽略了地面在多次散射中贡献的能量。四次散射项、五次散射项等均可以通过与上式相似的公式计算，只需要把其中的$L_2$对应地替换为$L_3, L_4$即可。

根据多次散射结果在空间中非常低频的特性，我们假设$L_2, L_3, L_4$等在点$p$的周围的取值可以被近似为$L_2(p), L_3(p), L_4(p)$等，于是积分内部的$L_i$可以被近似为$L_i(p)$，并提到外面来：

$$
\begin{aligned}
L_3(p) &= L_2(p)\int_{\mathcal S^2}\left(\int_0^DT(p, q)\sigma_s(q)dl_q\right)\rho_ud\omega \\
L_4(p) &= L_3(p)\int_{\mathcal S^2}\left(\int_0^DT(p, q)\sigma_s(q)dl_q\right)\rho_ud\omega \\
L_5(p) &= L_4(p)\int_{\mathcal S^2}\left(\int_0^DT(p, q)\sigma_s(q)dl_q\right)\rho_ud\omega \\
&\cdots\cdots
\end{aligned}
$$

这个近似有个很大的槽点——虽然说$L_i$在空间中确实变化缓慢，但将其视为一个常量也只应局限于某个小范围内；而上式中，$D$可能有非常大的值，其对应的$q$点离$p$点有数百公里也是很正常的，将这样的$q$处的$L_i$也近似为$L_i(p)$，我只能说效果好就必有其合理性了……

现令：

$$
f = \int_{\mathcal S^2}\left(\int_0^DT(p, q)\sigma_s(q)dl_q\right)\rho_ud\omega
$$

那么$L_3(p) = L_2(p)f, L_4(p) = L_2(p)f^2$。然后把上面的多次散射项都加起来：

$$
L_*(p) = L_2(p)(1 + f + f^2 + \cdots) = \frac{L_2(p)}{1 - f}
$$

由于忽略了地表起伏产生的遮蔽，根据球体的对称性，$L_2(p)$中的$p$实质上可以被简单地参数化为$p$的海拔高度$h_p$。再加上另一个参数——太阳光的方向相对于$p$的角度，我们就得到了另一张二维查找表$\mathbb M$。输入$p$的海拔高度和太阳角度，就能从表中查得单位太阳光强度下$p$处的$L_*$。

当然，我们不一定需要这么多近似才能计算出$\mathbb M$——在忽略地貌起伏的前提下，我们不需要略去地面反射，也不用假设$L_i$在局部可被当作常量，就可以用path tracing精确地计算出$\mathbb M$来。但上述方法提供了一种快速产生$\mathbb M$的技术，使得实时编辑大气参数成为了可能。

### 汇总

综上所述，$L(p\leftarrow\omega)$可由下式计算：

$$
\begin{aligned}
L(p\leftarrow\omega) &= L_eV_\text{sun}(p)\mathbb T(r_p, \theta_{p\omega}) &\text{太阳本身} \\
&+~T(p, x)f_s(\omega_\text{sun} \to x \to -\omega)E_eV_\text{sun}(x)\mathbb T(r_x, \theta_{x\omega_\text{sun}})  &\text{地表反射}\\
&+~\sum_q T(p, q)\sigma_s(h_q)\left(\rho(\cos\langle \omega_{\text{sun}}, \omega\rangle)E_e\mathbb T(r_q, \theta_{q\omega_\text{sun}})V_\text{sun}(q))+\mathbb M(h_q, \omega_\text{sun})\right)\Delta l_q &\text{单/多次散射}
\end{aligned}
$$

其中太阳本身作为一个“圆盘”，可以在大气绘制完成之后单独叠加上去；地表反射项可以在绘制地表时计算，单次散射和多次散射则需要通过一个ray marching过程计算。此外，有的透射率项被写成$\mathbb T$，表示它的值通过查表获得，而有的则被保留成$T(a, b)$的形式，这表示它的值是在ray marching时累积得到的。

## 实现

我用C++和DirectX 11实现了本文所述的大气渲染模型，代码仓库位于[AtmosphereRenderer](https://github.com/AirGuanZ/AtmosphereRenderer)。

### 低分辨率计算

由于天空颜色分布比较低频，可以计算一个低分辨率的天空纹理，然后在渲染时采样它，以此提高性能。在靠近地平线的角度，天空的颜色变化往往会比其他区域要剧烈一些，因此可以通过调整天空纹理坐标和方向的关系来改善其效果，譬如：

$$
v = 0.5\left(1 + \mathrm{sign}(l)\sqrt{\frac{|l|}{\pi/2}}\right)~~~~~(l \in [-\pi/2, \pi/2])
$$

其中$l$是方向向量与水平面的夹角。这个纹理只需要在太阳角度改变时重新计算，其计算代价也不高，即使每帧进行也不会带来很严重的负担。典型的天空纹理如下图所示：

<p align="center">
<img width="30%" height="30%" src="{{site.url}}/postpics/par/04.png">
</p>

在渲染场景中的物体时，我们需要从摄像机到物体表面进行ray marching，以计算这段路径上的积分。为了提高效率，可以预先将摄像机视锥体划分为一个较低分辨率的三维表格$A$，然后填充每个表项对应的位置与摄像机之间的透射率和内散射积分。在渲染场景中的物体时，直接查表即可。

在考虑了遮蔽项（$V_\text{sun}$）时，如果表格$A$的分辨率太低，其中的体积阴影容易产生“锯齿感”，如下图：

<p align="center">
<img width="50%" height="50%" src="{{site.url}}/postpics/par/02.png">
</p>

提高$A$的分辨率自然可以改善这一现象，但要将其完全掩盖，需要相对较高的分辨率，这会带来不小的计算开销。因此，我们退而求其次，抖动$A$的采样点位置来掩盖锯齿。将每个采样点在一定范围内随机抖动可得：

<p align="center">
<img width="50%" height="50%" src="{{site.url}}/postpics/par/03.png">
</p>

这把锯齿状的artifact转化成了噪声，更容易被人眼所忽略。此外，采用屏幕空间的蓝噪声可以起到更好的效果。

### 效果图

<p align="center">
<img src="{{site.url}}/postpics/par/gallery/00.png">
</p>
<p align="center">
<img src="{{site.url}}/postpics/par/gallery/01.png">
</p>
<p align="center">
<img src="{{site.url}}/postpics/par/gallery/02.png">
</p>
<p align="center">
<img src="{{site.url}}/postpics/par/gallery/03.png">
</p>

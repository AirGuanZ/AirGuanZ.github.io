---
title: A General Operator Formulation of Light Transport
key: t20181012
tags:
  - Graphics
---

<!--more-->

## 射线空间

设$\mathcal M$是无参与性介质（participating medium）的场景中的全体表面构成的集合，$\mathcal S^2$是全体单位方向向量，那么射线空间可以被定义为：

$$
\mathcal R = \mathcal M \times \mathcal S^2
$$

$\mathcal R$包含了所有从场景中的表面上发出的射线，其中的每个元素都可以被写成$(\boldsymbol x, \omega)$，$\boldsymbol{x}$为起点，$\omega$为方向。由于场景中不包含参与性介质，辐射（radiance）在任何一条射线上都保持不变，因此不必将场景中所有的点都作为射线的起点，只需要考虑场景表面的点即可。

**吞吐量测度**. 考虑射线$\boldsymbol{r} = (\boldsymbol{x}, \omega)$附近的一小束射线，它们占据了$dA$的区域和$d\omega$的立体角，这一小束射线的吞吐量（throughput）被定义为：

$$
d\mu(\boldsymbol{r}) = d\mu(\boldsymbol{x}, \omega) = dA(\boldsymbol{x})d\omega^{\perp}_{\boldsymbol{x}}(\omega) = dA^{\perp}(\boldsymbol{x})d\omega_{\boldsymbol{x}}(\omega)
$$

对于$\mathcal R$的射线集合$D$，$\mu(D)$可以被定义为：

$$
\mu(D) = \int_DdA(\boldsymbol{x})d\omega^{\perp}_{\boldsymbol{x}}(\omega)
$$

## 散射与传播算子

从物理的角度来看，光的传播可以被描述为两个“步骤”：一是散射，即光子和表面产生交互的过程；二是传播，即光子在一种确定介质中沿直线“移动”的过程。这两个步骤可以被辐射度函数上的线性算子描述：

**局部散射算子**. 定义局部散射算子（local scattering operator）为：

$$
(Kh)(\boldsymbol{x}, \omega_o) = \int_{\mathcal S^2}f_s(\boldsymbol{x}, \omega_i \to \omega_o)h(\boldsymbol{x}, \omega_i)d\omega^{\perp}_{\boldsymbol{x}}(\omega_i)
$$

对任意一个入射辐射函数$L_i$，$K$给出一个出射辐射函数$L_o = KL_i$。

**传导算子**. 用$\boldsymbol{x}_{\mathcal M}(\boldsymbol{x}, \omega)$表示光线求交函数，即先令

$$
d_{\mathcal M}(\boldsymbol{x}, \omega) = \inf\{d > 0\mid \boldsymbol{x} + d\omega \in \mathcal M\}
$$

为表面距离函数，于是光线求交可以被定于为：

$$
\boldsymbol{x}_{\mathcal M}(\boldsymbol{x}, \omega) = \boldsymbol{x} + d_{\mathcal M}(\boldsymbol{x}, \omega)\omega
$$

它表示从$\boldsymbol{x}$沿$\omega$出发第一个可见的$\mathcal M$中的点。基于此，传导算子（propagation operator）可以被定义为：

$$
(Gh(\boldsymbol{x}, \omega_i)) = \begin{cases}
    h(\boldsymbol{x}_{\mathcal M}(\boldsymbol{x}, \omega_i), -\omega_i) \text{ if }d_{\mathcal M}(\boldsymbol{x}, \omega_i) < \infty \\
    0 \text{ otherwise}
\end{cases}
$$

传导算子把入射辐射函数$L_i$用初涉复设函数来表示，即$L_i = GL_o$。

**传播算子**. 令$T = KG$，$T$被称为光线传播算子（light transport operator）,它把出射辐射函数$L_o$变换为一次散射后的$TL_o$。光线传播方程也可以用它来改写：

$$
L = L_e + TL
$$

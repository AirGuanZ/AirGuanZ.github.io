---
title: Weingarten Equations
key: t20180911
tags:
  - Graphics
  - Mathematics
---

<!--more-->

设$S$是三维欧氏空间中的一个参数曲面，$\boldsymbol p = P(u, v)$是$S$上的一个点，$\boldsymbol n$是$S$上的单位法向量，则：

$$
\begin{aligned}
\frac{\partial \boldsymbol n}{\partial u} &=
    \frac{FM - GL}{EG - F^2}\frac{\partial \boldsymbol p}{\partial u} +
    \frac{FL - EM}{EG - F^2}\frac{\partial \boldsymbol p}{\partial v} \\
\frac{\partial \boldsymbol n}{\partial v} &=
    \frac{FN - GM}{EG - F^2}\frac{\partial \boldsymbol p}{\partial u} +
    \frac{FM - EN}{EG - F^2}\frac{\partial \boldsymbol p}{\partial v}
\end{aligned}
$$

其中$E, F, G$是$S$的first fundamental form的系数，$L, M, N$是其second fundamental form的系数：

$$
\begin{aligned}
E &= \left| \frac{\partial \boldsymbol p}{\partial u} \right|^2 \\
F &= \frac{\partial \boldsymbol p}{\partial u} \cdot \frac{\partial \boldsymbol p}{\partial v} \\
G &= \left| \frac{\partial \boldsymbol p}{\partial v} \right|^2 \\
L &= \boldsymbol n \cdot \frac{\partial^2\boldsymbol p}{\partial u^2} \\
M &= \boldsymbol n \cdot \frac{\partial^2\boldsymbol p}{\partial u\partial v} \\
N &= \boldsymbol n \cdot \frac{\partial^2\boldsymbol p}{\partial v^2}
\end{aligned}
$$

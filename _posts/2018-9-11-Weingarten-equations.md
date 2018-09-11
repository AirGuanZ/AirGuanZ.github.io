---
layout: post
title: Weingarten Equations
key: t20180911
tags:
  - Graphics
  - Mathematics
---

<!--more-->

设$S$是三维欧氏空间中的一个参数曲面，$\bm p = P(u, v)$是$S$上的一个点，$\bm n$是$S$上的单位法向量，则：

$$
\begin{aligned}
\frac{\partial \bm n}{\partial u} &=
    \frac{FM - GL}{EG - F^2}\frac{\partial \bm p}{\partial u} +
    \frac{FL - EM}{EG - F^2}\frac{\partial \bm p}{\partial v} \\
\frac{\partial \bm n}{\partial v} &=
    \frac{FN - GM}{EG - F^2}\frac{\partial \bm p}{\partial u} +
    \frac{FM - EN}{EG - F^2}\frac{\partial \bm p}{\partial v}
\end{aligned}
\begin{aligned}
\frac{\partial \bm n}{\partial u} &=
    \frac{Ff - Ge}{EG - F^2}\frac{\partial \bm p}{\partial u} +
    \frac{Fe - Ef}{EG - F^2}\frac{\partial \bm p}{\partial v} \\
\frac{\partial \bm n}{\partial v} &=
    \frac{Fg - Gf}{EG - F^2}\frac{\partial \bm p}{\partial u} +
    \frac{Ff - Eg}{EG - F^2}\frac{\partial \bm p}{\partial v}
\end{aligned}
$$

其中$E, F, G$是$S$的first fundamental form的系数，$L, M, N$是其second fundamental form的系数：

$$
\begin{aligned}
E &= \left| \frac{\partial \bm p}{\partial u} \right|^2 \\
F &= \frac{\partial \bm p}{\partial u} \cdot \frac{\partial \bm p}{\partial v} \\
G &= \left| \frac{\partial \bm p}{\partial v} \right|^2 \\
L &= \bm n \cdot \frac{\partial^2\bm p}{\partial u^2} \\
M &= \bm n \cdot \frac{\partial^2\bm p}{\partial u\partial v} \\
N &= \bm n \cdot \frac{\partial^2\bm p}{\partial v^2}
\end{aligned}
$$

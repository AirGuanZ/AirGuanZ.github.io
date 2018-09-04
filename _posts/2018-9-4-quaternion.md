---
layout: post
title: 四元数基础
key: 20180904
tags:
  - Graphics
---

学了这么久的graphics，连四元数都迷迷糊糊的，索性花点时间相关东西理清楚。

<!--more-->

## 复数与二维旋转

二维情形下，将某个点$(x, y)$绕原点$(0, 0)$逆时针旋转$\theta$的变换矩阵为：

$$
\left[\begin{matrix}
    \cos\theta & -\sin\theta \\
    \sin\theta & \cos\theta
\end{matrix}\right]
\left[\begin{matrix}
    x \\ y
\end{matrix}\right]
=
\left[\begin{matrix}
    x\cos\theta - y\sin\theta \\
    x\sin\theta + y\cos\theta
\end{matrix}\right]
$$

也可以利用复数乘法的几何解释来完成旋转：

$$
e^{i\theta}(x + iy)
= (\cos\theta + i\sin\theta)(x + iy)
= (\cos\theta - y\sin\theta) + i(x\sin\theta + y\cos\theta)
$$

籍由这两种旋转表示方式的等价性，可以将其拓展到三维空间。

## 绕任意轴的三维旋转公式

现需要在右手系下将向量$\vec a$绕轴$\vec d$旋转$\theta$度，显然$\vec a$可以被分解为平行于$\vec d$和垂直于$\vec d$的两个分量，其中前者无需修改：

$$
\begin{aligned}
& \vec a = \vec a_{\perp} + \vec a_{\parallel} \\
& \vec a_{\prep} = \vec a - \vec a_{\parallel} \\
& \vec a_{\parallel} = \mathrm{Proj}_{\vec d}\vec a
\end{aligned}
$$

若将$\vec e_{\vec a_{\prep}}$视为$x'$轴，将$\vec e_{\vec d}$视为$z'$轴，则求它们的叉积即可得到$y'$轴。此时只需要将经典的二维旋转变换应用到$x'y'z'$坐标系中的$x'y'$平面上，即可求出旋转后的$\vec a_{\prep}'$，也就求出了$\vec a' = \vec a_{\prep}' + \vec a_{\parallel}$。设$L_p = \vert{\vec a_{\prep}}$，则$\vec a_{\prep}$在$x'y'z'$中的坐标为$(L_p, 0, 0)$，此时在$x'y'$平面上进行旋转变换，得到：

$$
\vec a_{\prep}' =
\left[\begin{matrix}
    \vec e_{x'} & \vec e_{y'}
\end{matrix}\right]
\left(
\left[\begin{matrix}
    \cos\theta & -\sin\theta \\
    \sin\theta & \cos\theta
\end{matrix}\right]
\left[\begin{matrix}
    L_p \\ 0
\end{matrix}\right]
\right)
$$

待续……

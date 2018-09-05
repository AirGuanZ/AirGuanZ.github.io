---
layout: post
title: 四元数基础
key: 20180904
tags:
  - Graphics
  - Mathematics
---

学了这么久的graphics，居然连四元数都迷迷糊糊的，真是丢人急先锋，索性花点时间相关东西理清楚。

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

## 三维旋转公式

现需要在右手系下将向量$\vec a$绕单位向量$\vec d$旋转$\theta$度，显然$\vec a$可以被分解为平行于$\vec d$和垂直于$\vec d$的两个分量，其中前者无需修改：

$$
\begin{aligned}
& \vec a = \vec a_{\perp} + \vec a_{\parallel} \\
& \vec a_{\perp} = \vec a - \vec a_{\parallel} \\
& \vec a_{\parallel} = \mathrm{Proj}_{\vec d}\vec a = (\vec a\cdot\vec d)\vec d
\end{aligned}
$$

若将$\vec e_{\vec a_{\perp}}$视为$x'$轴，将$\vec d$视为$z'$轴，则求它们的叉积即可得到$y'$轴。此时只需要将经典的二维旋转变换应用到$x'y'z'$坐标系中的$x'y'$平面上，即可求出旋转后的$\vec a_{\perp}'$，也就求出了$\vec a' = \vec a_{\perp}' + \vec a_{\parallel}$。设$L_p = \vert\vec a_{\perp}\vert$，则$\vec a_{\perp}$在$x'y'z'$中的坐标为$(L_p, 0, 0)$，此时在$x'y'$平面上进行旋转变换，得到：

$$
\begin{aligned}
    \vec a_{\perp}'
    &=
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
    \right) \\
    &=
    \cos\theta \vec a_{\perp} + \sin\theta\left(\vec d \times \vec a_{\perp}\right) \\
    &= \cos\theta\vec a_{\perp} + \sin\theta\left(\vec d \times \vec a\right)
\end{aligned}
$$

从而旋转后的$\vec a'$为

$$
\vec a'
= \vec a_{\perp} + \vec a_{\parallel}'
= (1 - \cos\theta)(\vec a\cdot \vec d)\vec d + \cos\theta\vec a + \sin\theta(\vec d \times \vec a)
$$

## 四元数

顾名思义，四元数包含一个实部和三个虚部，虚部单位分别以$i, j, k$表示，它们满足：

$$
ij = k,~jk = i,~ki = j,~i^2 = j^2 = k^2 = -1
$$

四元数被定义为$1, i, j, k$的线性组合（没错，四元数定义到这就算结束了），加法减法乘法共轭以及范数都可以直接拓展复数的相关定义得到：

$$
\begin{aligned}
(a+ib+jc+kd) + (e+if+jg+kh) &= (a+e) + i(b+f) + j(c+g) + k(d+h) \\
(a+ib+jc+kd) - (e+if+jg+kh) &= (a-e) + i(b-f) + j(c-g) + k(d-h) \\
(a+ib+jc+kd) \times (e+if+jg+kh) &= (ae-bf-cg-dh) + i(be+af-dg+ch) \\
&+ j(ce+df+ag-bh) + k(de-cf+bg+ag) \\
(a+ib+jc+kd)^* &= a-ib-jc-kd \\
\vert a+ib+jc+kd \vert &= \sqrt{a^2 + b^2 + c^2 + d^2}
\end{aligned}
$$

四元数$a + ib + jc + kd$常被记作$[a, \vec v]$，其中：

$$
\vec v = \left[\begin{matrix}
    b \\ c \\ d
\end{matrix}\right]
$$

可以验证：

$$
\begin{aligned}
[s, \vec v]\times [t, \vec u] &= [st - \vec v\cdot \vec u, s\vec u + t\vec v + \vec v\times \vec u] \\
[s, \vec v] + [t, \vec u] &= [s + t, \vec v + \vec u] \\
[s, \vec v]^* &= [s, -\vec v]^* \\
\vert [s, \vec v]^* \vert &= \vert [s, \vec v] \vert
\end{aligned}
$$

最后来个逆：四元数$q$的逆是指满足$qq^{-1} = q^{-1} = 1$的四元数$q^{-1}$。根据$q^*$和$q$的长度关系，有：

$$
\dfrac{qq^*}{\vert q\vert^2} = 1 \Rightarrow q^{-1} = \dfrac{q^*}{\vert q\vert^2}
$$

特别地，单位四元数的逆就是其共轭。

## 四元数和三维旋转

根据之前的讨论，有：

$$
\vec a_\perp' = \cos\theta\vec a_\perp + \sin\theta(\vec d\times \vec a_\perp)
$$

若令四元数$a_\perp = [0, \vec a_\perp], d = [0, \vec d]$，则$da_\perp = [0, \vec d\times\vec a_\perp]$，于是$a_\perp' = [0, \vec a_\perp']$满足：

$$
a_\perp' = [0, \cos\theta\vec a_\perp + \sin\theta(\vec d\times \vec a_\perp)] = \cos\theta a_\perp + \sin\theta(da_\perp) = (\cos\theta+\sin\theta d)a_\perp
$$

可见这一旋转可以认为是将四元数$cos\theta+\sin\theta d$作用到$a_\perp$上得到的。

验证可知若$\vec d$为单位向量，则$[\cos\theta, \sin\theta\vec d]^2 = [cos(2\theta), \sin(2\theta)\vec d]$，其意义很直观——如果把单位四元数$q$视为一个旋转变换，那么$q^2$相当于进行两次这样的变换，即将旋转角翻倍。基于此，若令：

$$
\begin{aligned}
a' &= [0, \vec a] \\
a_\parallel &= [0, \vec a_\parallel] \\
p &= [\cos(\frac 1 2\theta), \sin(\frac 1 2\theta)\vec d]
\end{aligned}
$$

则有：

$$
a' = a_\parallel + ppa_\perp = pp^{-1}a_\parallel + ppa_\perp = pp^*a_\parallel + ppa_\perp
$$

容易验证$pa_\parallel = a_\parallel p$，$pa_\perp = a_\perp p^*$，代入上式得：

$$
a' = pa_\parallel p^* + pa_\perp p^* = p(a_\parallel + a_\perp)p^* = pap^*
$$

这就得到了用四元数进行绕任意轴旋转的公式。

## 球面线性插值

我们通常用旋转矩阵来进行三维空间中的旋转操作，但这在需要对旋转角进行插值时（比如涉及到旋转的动画）并不好用，因为这不能通过直接对矩阵元素进行线性插值来实现。四元数则提供了一种方便的插值方式——由于旋转角$\theta$在四元数中直接出现，因此可以把将$\theta$线性插值的四元数用于旋转操作，这将提供一个平滑的旋转过程，称为球面线性插值（Spherical Linear Interpolation，Slerp）。

设初始旋转四元数是$q$，结束旋转四元数是$q'$，插值系数为$t \in [0, 1]$，若令：

$$
\Delta q = q' \Rightarrow \Delta = q'q^{-1} = q'q^*
$$

则$\Delta$也是一个旋转四元数，它必然具有$[\cos(\frac 1 2\theta), \sin(\frac 1 2\theta)\vec d]$的形式，此时对$\Delta$的旋转角在$[0, \frac 1 2\theta]$间进行线性插值即可，即：

$$
q(t) = \left[\cos\left(\frac t 2\theta\right), \sin\left(\frac t 2\theta\right)\vec d\right]
$$

在实际应用中，Slerp有个小坑——朝某个方向旋转$\theta$角度和朝它的反方向旋转$2\pi - \theta$在结果上是等价的，在插值过程中却表现得完全不同，因此，若$q$和$q'$间得夹角超过$\pi$，即$q\cdot q' < 0$，则有必要将$q'$换成$-q'$后再进行插值。

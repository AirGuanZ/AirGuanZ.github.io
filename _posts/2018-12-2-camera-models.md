---
layout: post
title: 摄像机模型小结
key: t20181202
tags:
  - Graphics
---

最近在试着实现light tracing算法，需要手推各种摄像机模型的measurement function。除了最简单的正交和透视投影摄像机外，我居然一个都推不出来……感觉是自己对各种摄像机模型的理解不够，因此重新学习和思考一番，总结在这里。这篇文章内容是比较trivial的，只是我需要把它们完全理清楚才好实现。

本文一律使用右手坐标系，且视$z$轴正方向为垂直向上的方向。此外，本文的讨论并不立足于实时渲染，因此不会涉及到透视投影变换、齐次坐标系等话题。

<!--more-->

## 小孔摄像机

顾名思义，小孔摄像机模型中，光只能通过一个理论上大小可以忽略的孔洞进入摄像机内部，并命中成像平面上的感光元器件。我们假设这个布满感光元器件的接收设备是一个矩形，且在世界坐标系中宽度为$w_S$，高度为$h_S$，并以它的几何中心为原点，在该平面上建立一个简单的二维坐标系，用来表示元器件平面上的位置。这一坐标系被称为感光元器件坐标系，记作$C_S$，其有效的坐标取值范围是（注意像素下标虽然是整数，但像素坐标系中的点未必是）：

$$
(x, y) \in \left[-\frac{w_S}{2}, \frac{w_S}{2}\right] \times \left[-\frac{h_S}{2}, \frac{h_S}{2}\right]
$$

元器件所接收到的信号最后要转换为一副二维图像上的颜色。设图像宽高分别为$w_I$像素和$h_I$像素，以左上角为原点，则有效的像素坐标范围为$[0, w_I] \times [0, h_I]$。我们把这个坐标系称为像素坐标系，记作$C_I$。$C_I$和$C_S$间的关系是非常明显的：

$$
\begin{aligned}
    C_I \Rightarrow C_S:~~~~~~~&\begin{cases}
        x_S = w_S(x_I/w_I) - w_S/2 \\
        y_S = h_S/2 - h_S(y_I/w_I)
    \end{cases} \\
    C_S \Rightarrow C_I:~~~~~~~&\begin{cases}
        x_I = (x_S / w_S + 0.5) * w_I \\
        y_I = (0.5 - y_S / h_S) * h_I
    \end{cases}
\end{aligned}
$$

除了$C_I$和$C_S$间的转换，我们还需要把$C_S$中的坐标转换为世界坐标系中的射线的能力。为了简化问题，我们先考虑一个本地坐标系$C_L$中的情况：摄像机的小孔位于原点，正面朝向$\vec e_x$方向，小孔和元器件平面的距离为$L$，则给定$C_S$中的坐标$(x_S, y_S)$，$C_L$中的射线原点和方向就可以表示为：

$$
\begin{aligned}
    \mathrm{origin} &= [0, 0, 0]^T \\
    \mathrm{direction} &= [L, x_S, -y_S]^T
\end{aligned}
$$

最后，我们引入图形学中非常常见的本地坐标系$C_L$与世界坐标系$C_W$间的变换。假设摄像机可以先被旋转，再被平移。令$\vec d$表示摄像机旋转后的正面朝向，$\vec u$为$C_L$中的$xOz$平面上任意一个不平行于$\vec e_x$的单位向量被变换到在$C_W$中的结果，则$C_L$中的$\vec e_x, \vec e_y, \vec e_z$在$C_W$中分别对应：

$$
\begin{aligned}
    \vec e_x &\Rightarrow \vec d \\
    \vec e_y &\Rightarrow \vec u \times \vec e_x \\
    \vec e_z &\Rightarrow \vec e_x \times \vec e_y
\end{aligned}
$$

由于旋转是个线性变换，所以可以用一个矩阵$\boldsymbol R_{L \to W}$来表示，且$\boldsymbol R_{L \to W}$可以被构造为：

$$
\boldsymbol R_{L \to W} = \left[\begin{matrix}
    \vec d & \vec u \times \vec d & \vec d \times \left(\vec u \times \vec d\right)
\end{matrix}\right]
$$

再加上一个平移偏移量$\vec T$，我们就得到了$C_L$和$C_W$间的转换方法：

$$
\begin{aligned}
    C_L \to C_W:~~~~~~~&\begin{cases}
        \vec v_W = \boldsymbol R\vec v_L \\
        \vec p_W = \boldsymbol R\vec p_L + \vec T
    \end{cases} \\
    C_W \to C_L:~~~~~~~&\begin{cases}
        \vec v_L = \boldsymbol R^T\vec v_W \\
        \vec p_L = \boldsymbol R^T\left(\vec p_W - \vec T\right)
    \end{cases}
\end{aligned}
$$

其中$\vec v$表示方向，$\vec p$表示位置；$\boldsymbol R^T$表示的其实是$\boldsymbol R^{-1}$，因为旋转矩阵是单位正交矩阵，所以可以通过转置来快速求它的逆。

## Camera Measure

在讨论基于光线追踪的真实感渲染技术时，我们总是说：只要对任意点$\vec p$和方向$\vec \omega$能够求解出$L(\vec x \leftarrow \vec \omega)$，就算是大功告成了。但是仔细一想：图像上的每个像素实际上都对应了场景中的一小块区域，$L(\vec x \leftarrow \vec \omega)$和该像素的颜色间是怎么建立起联系的呢？

设像素$j$在场景中对应的元器件区域为$\mathcal M_j$，对每个$\vec x \in \mathcal M_j$和方向$\vec \omega \in \mathcal S^2$，$\vec x$处的元器件对来自$\vec \omega$的光都会产生一定的响应，并影响到最终我们观察到的$j$的颜色。设该响应的强度可以用函数$W_e^{(j)}(\vec x \to \vec \omega)$表示，那么像素$j$的颜色为：

$$
I_j = \int_{\mathcal M_j}\int_{\mathcal S^2}W_e^{(j)}(\vec x \to \vec \omega)L(\vec x \leftarrow \vec \omega)\cos\langle \vec N_x, \vec \omega\rangle d\omega dA_x
$$

以小孔摄像机为例，如果我们简单地认为像素$j$的颜色为在$\mathcal M_j$意义上通过小孔击中$\mathcal M_j$的所有radiance的平均值，那么$W_e^{(j)}$为：

$$
W_e^{(j)}(\vec x \to \vec \omega) =
\frac{
    \delta\left(
        \vec \omega -
        \mathrm{Nor}\left(
            \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
        \right)
    \right)
}{
    \cos\langle\vec N_x, \vec\omega\rangle\int_{\mathcal M_j}dA
}
$$

其中$\mathrm{Nor}$表示将向量归一化。这个式子的意思是对$\mathcal M_j$上的每个点$\vec x$，只有从小孔所在的方向来的光可以对$j$的颜色作出贡献。当然，我们也可以把$\cos$因子纳入考虑，使得结果更加严格：

$$
\begin{aligned}
W_e^{(j)}(\vec x \to \vec \omega) &= \frac{\delta\left(\vec \omega - \mathrm{Nor}\left(
            \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
        \right)\right)}{\int_{\mathcal M_j}\int_{\mathcal S^2} \delta\left(\vec \omega - \mathrm{Nor}\left(
            \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
        \right)\right)\cos\langle\vec N_x, \vec\omega\rangle d\omega dA} \\
&= \frac{
    \delta
    \left(
        \vec \omega - \mathrm{Nor}\left(
            \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
        \right)
    \right)
}{
    \int_{\mathcal M_j}
    \cos\langle
        \vec N_x,
        \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
    \rangle
    dA
} \\
&= \frac{
    \delta
    \left(
        \vec \omega - \mathrm{Nor}\left(
            \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
        \right)
    \right)
}{
    \int_{\mathcal M_j}
    L / \sqrt{L^2 + x_{S_j}^2 + y_{S_j}^2}
    dA
}
\end{aligned}
$$

## 简化的薄凸透镜摄像机



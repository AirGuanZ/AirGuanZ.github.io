---
layout: post
title: 摄像机模型小结
key: t20181202
tags:
  - Atrc
  - Graphics
---

本文内容已作废，原因参见[此文](http://alvyray.com/Memos/CG/Microsoft/6_pixel.pdf)。我最近在重构Atrc，没有再采用本文对摄像机的建模方法，尤其是measurement equation的形式。

<!--more-->

最近在试着实现light tracing算法，需要手推各种摄像机模型的measurement function，打算把结果总结在这里。这篇文章内容是比较trivial的，只是需要把它们完全理清楚才好实现。

本文一律使用右手坐标系，且视$z$轴正方向为垂直向上的方向。此外，本文的讨论并不立足于实时渲染，因此不会涉及到透视投影变换、齐次坐标系等话题。

所有内容的实现均可在[我的GitHub](https://github.com/AirGuanZ/Atrc/tree/master/Source/Atrc/Camera)上找到。

## Pinhole Camera

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

除了$C_I$和$C_S$间的转换，我们还需要把$C_S$中的坐标转换为世界坐标系中的射线的能力。为了简化问题，我们先考虑一个本地坐标系$C_L$中的情况：摄像机的小孔位于原点，正面朝向$\vec e_x$方向，小孔和元器件平面的距离为$L$，则给定$C_S$中的坐标$(x_S, y_S)$，$C_L$中的射线原点和方向可以表示为：

$$
\begin{aligned}
    \mathrm{origin} &= [0, 0, 0]^T \\
    \mathrm{direction} &= [L, x_S, -y_S]^T
\end{aligned}
$$

最后，我们引入图形学中非常常见的本地坐标系$C_L$与世界坐标系$C_W$间的变换。假设摄像机可以先被旋转，再被平移。令$\vec d$表示摄像机旋转后的正面朝向，$\vec u$为$C_L$中的$xOz$平面上任意一个不平行于$\vec e_x$的单位向量被变换到$C_W$中的结果，则$C_L$中的$\vec e_x, \vec e_y, \vec e_z$在$C_W$中分别对应：

$$
\begin{aligned}
    \vec e_x &\Rightarrow \vec d \\
    \vec e_y &\Rightarrow \vec u \times \vec d \\
    \vec e_z &\Rightarrow \vec d \times (\vec u \times \vec d)
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

其中$\vec v$表示方向，$\vec p$表示位置；$\boldsymbol R^T$表示的其实是$\boldsymbol R^{-1}$，因为旋转矩阵是单位正交矩阵，所以可以通过转置来快速求它的逆。$C_L$和$C_W$间的转换方法适用于后文中所有的摄像机模型。

## Measurement Function

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
    \cos\langle\vec N_x, \vec\omega\rangle\mathrm{area}(\mathcal M_j)
}
$$

其中$\mathrm{Nor}$表示将向量归一化。这个式子的意思是对$\mathcal M_j$上的每个点$\vec x$，只有从小孔所在的方向来的光可以对$j$的颜色作出贡献。

把这一节和上一节的内容结合起来，就可以实现小孔摄像机模型了。随便画个图看看：

![PICTURE]({{site.url}}/postpics/Atrc/30_2018_12_05_DoF_Comp.png)

其实它和图形学中常用的透视投影摄像机是等价的。此外，小孔摄像机拍摄的图像上下左右都颠倒了，为了方便展示，我把它翻转了回来。下文中展示的图像也做了类似的必要处理。

## Thin Lens Camera

理想的小孔摄像机在现实中是不存在的，因为光没法通过一个没有大小的孔洞来照亮整个传感元器件平面。本节讨论一种简化的凸透镜摄像机模型，它假设透镜的厚度可以被忽略。

考虑将一个薄凸透镜放置在数轴的原点处，一束平行光沿着$-\vec e_x$方向击中透镜，它们将汇聚于$-x$轴上的一点，该点和原点间的距离称为薄凸透镜的焦距，记作$f$。对位于距原点$z$（$z > f$）的物体，它所发出的光穿过凸透镜后也会汇聚于一点，设该汇聚点距原点为$z'$（$z' < f$），则$z$、$z'$、$f$满足以下关系：

$$
\frac 1 {z'} - \frac 1 z = \frac 1 f
$$

也就是说，给定元器件平面和透镜的距离$z'$，我们可以通过上式计算出$z$来。在三维空间中，这样一个和透镜中心距离为$z$的平面被称为焦平面，位于焦平面上的物体上的每个点所发出的光都将汇聚到元器件平面上的一点，而不在焦平面上的点所发出的光在穿过透镜后则会汇聚到元器件平面上的一个圆内，若该圆的大小超过了一个像素，那么我们将观察到模糊的物体，也就是所谓的景深效果。

简单起见，给定元器件平面上的点$\vec x$，我们认为薄凸透镜上的每一点$\vec y$都会对$\vec x$所响应的亮度作出同等的贡献，而每个$\vec x \in \mathcal M_j$也都对像素$j$的颜色作出同等的贡献。此外，我们也希望$W_e^{(j)}$能够满足归一化约束：

$$
\int_{\mathcal M_j}\int_{\mathcal S^2}W_e^{(j)}(\vec x \to \vec \omega)\cos\langle \vec N_x, \vec \omega\rangle d\omega dA_x = 1
$$

为此，我们首先把$\mathcal S^2$改写到$\mathcal M_A$上，其中$\mathcal M_A$是薄凸透镜在场景中对应的区域：

$$
\int_{\mathcal M_j}\int_{\mathcal M_A}W_e^{(j)}(\vec x \to \vec e_{xy})\cos\langle \vec N_x, \vec e_{xy}\rangle\frac{\cos\langle \vec N_y, \vec e_{yx}\rangle}{\left|\vec x - \vec y\right|_2^2} dA_y dA_x = 1
$$

由于每个$\vec y$都对$j$的颜色作出同等的贡献，因而：

$$
W_e^{(j)}(\vec x \to \vec e_{xy})\cos\langle \vec N_x, \vec e_{xy}\rangle\frac{\cos\langle \vec N_y, \vec e_{yx}\rangle}{\left|\vec x - \vec y\right|_2^2} = C
$$

$C$表示某个和$\vec y$无关的数。代入归一化条件后我们可以得到：

$$
\begin{aligned}
&\int_{\mathcal M_j}\int_{\mathcal M_A}C dA_y dA_x = 1 \\
\Rightarrow~&C = \frac 1 {\mathrm{area}(\mathcal M_j)\mathrm{area}(\mathcal M_A)}
\end{aligned}
$$

这就成功解出了$W_e^{(j)}$：

$$
W_e^{(j)}(\vec x \to \vec e_{xy}) =  \frac{\left|\vec x - \vec y\right|_2^2}{\cos\langle \vec N_x, \vec e_{xy}\rangle\cos\langle \vec N_y, \vec e_{yx}\rangle\mathrm{area}(\mathcal M_j)\mathrm{area}(\mathcal M_A)}
$$

容易验证以下三个关系的正确性：

$$
\begin{aligned}
    \cos\langle \vec N_y, \vec e_{yx}\rangle &= \cos\langle \vec N_x, \vec e_{xy}\rangle \\
    \left|\vec x - \vec y\right|_2^2 &= \frac{L^2}{\cos^2\langle \vec N_x, \vec e_{xy}\rangle} \\
    \mathcal M_A &= \pi r^2
\end{aligned}
$$

其中$L$是元器件平面和凸透镜间的距离，$r$是凸透镜半径。把这三个式子代入上面的$W_e^{(j)}$表达式中，再把$\vec N_x$换成与之相等的摄像机朝向$\vec d$，就得到了：

$$
W_e^{(j)}(\vec x \to \vec \omega) = \frac{L^2}{\pi r^2 \cos^4\langle \vec d, \vec \omega\rangle \mathrm{area}(\mathcal M_j)}
$$

凸透镜半径取得越大，不在焦平面上的物体显得越模糊。下图的元器件平面宽度为2m，距离透镜1m，透镜半径为20cm（这参数已经超出我对摄像机的认知了，不过请不要在意这些细节），可以看到景深效果非常夸张：

![PICTURE]({{site.url}}/postpics/Atrc/29_2018_12_05_SimpleDoF.png)

## Environment Camera

环境摄像机不是现实中能有的摄像机——它捕获四面八方的光线，将上下左右前后所有的景物都记录到一幅图像上。对任意一个入射方向$\omega$，设$\theta_\omega \in [0, 2\pi)$是它沿垂直轴的旋转角度，$\phi_\omega \in [-\pi/2, \pi/2]$是与水平面的夹角。令：

$$
\begin{aligned}
    u_\omega &= 1-\frac{\theta_\omega}{2\pi} \\
    v_\omega &= 0.5 - \frac{\phi_\omega}{\pi}
\end{aligned}
$$

并将$(u_\omega, v_\omega)$作为纹理坐标（原点在图像左上角），就能获得一幅描述了环境光照的图像。

环境摄像机的感光元器件就比较邪门儿了，它是一个球。我们暂且把这个球放置在原点，用$r$表示其半径，则$(u_\omega, v_\omega)$对应球面上的点为：

$$
\begin{cases}
    x_\omega = r\cos\phi_\omega\cos\theta_\omega \\
    y_\omega = r\cos\phi_\omega\sin\theta_\omega \\
    z_\omega = r\sin\phi_\omega
\end{cases}
$$

这个点上的感光器件只对一个方向来的光有响应，那就是来自$[x, y, z]^T$的光。于是，给定像素坐标系$C_I$中的点$(x_I, y_I)$，其对应的射线是：

$$
\begin{aligned}
    &\mathrm{origin}(x_I, y_I) = \left[\begin{matrix}
        r\cos\left(\frac{\pi}{2} - \frac{\pi y_I}{h_I}\right)\cos\left(-\frac{2\pi x_I}{w_I}\right) \\
        r\cos\left(\frac{\pi}{2} - \frac{\pi y_I}{h_I}\right)\sin\left(-\frac{2\pi x_I}{w_I}\right) \\
        r\sin\left(\frac{\pi}{2} - \frac{\pi y_I}{h_I}\right)
    \end{matrix}\right]
    = \left[\begin{matrix}
        r\sin\frac{\pi y_I}{h_I}\cos\left(-\frac{2\pi x_I}{w_I}\right) \\
        r\sin\frac{\pi y_I}{h_I}\sin\left(-\frac{2\pi x_I}{w_I}\right) \\
        r\cos\frac{\pi y_I}{h_I}
    \end{matrix}\right] \\
    &\mathrm{direction}(x_I, y_I) = \vec N_\mathrm{origin} = \frac{\mathrm{origin}}{\left|\mathrm{origin}\right|_2}
    = \left[\begin{matrix}
        \sin\frac{\pi y_I}{h_I}\cos\left(-\frac{2\pi x_I}{w_I}\right) \\
        \sin\frac{\pi y_I}{h_I}\sin\left(-\frac{2\pi x_I}{w_I}\right) \\
        \cos\frac{\pi y_I}{h_I}
    \end{matrix}\right]
\end{aligned}
$$

设像素$j$在$C_I$中的范围为$[x_{j_1}, x_{j_2}] \times [y_{j_1}, y_{j_2}]$，则它所对应的元器件区域$\mathcal M_j$为：

$$
\mathcal M_j = \{ \mathrm{origin}(x, y) \mid (x, y) \in [x_{j_1}, x_{j_2}] \times [y_{j_1}, y_{j_2}] \}
$$

我们假设$\mathcal M_j$上的每个点对$j$的颜色有相同的贡献量，于是：

$$
\begin{aligned}
    W_e^{(j)}(\vec x \to \vec \omega) = C\frac{\delta\left(\vec N_x - \vec \omega\right)}{\cos\langle \vec N_x, \vec \omega\rangle} \\
    \int_{\mathcal M_j}\int_{\mathcal S^2}W_e^{(j)}(\vec x \to \vec \omega)\cos\langle \vec N_x, \vec\omega\rangle d\omega dA_x = 1
\end{aligned}
$$

解得：

$$
W_e^{(j)}(\vec x \to \vec \omega) =  \frac{\delta\left(\vec N_x - \vec \omega\right)}{\mathrm{area}(\mathcal M_j)}
$$

随便画个图看看：

![PICTURE]({{site.url}}/postpics/Atrc/2018_12_06_EnvironmentCamera.png)

## Measurement Equation on Film

对环境摄像机的探究就算是大功告成了吗？其实不然。在之前讨论的两个摄像机模型中，如果我们在像素$j$中均匀采样像素平面上的点，那么这也等价于在$\mathcal M_j$上均匀采样，进而可以通过均匀采样和求均值来估计$I_j$。而在环境摄像机模型中，像素$j$上的均匀分布并不等价于$\mathcal M_j$上的均匀分布，因而在$C_I$中，像素$j$范围内的各点对$I_j$的贡献是不等的。我们把measurement equation中的$\mathcal M_j$换到$C_I$上试试：

$$
I_j = \int_{x_0}^{x_1}\int_{y_0}^{y_1}\int_{\mathcal S^2}Q_e^{(j)}((x, y) \to \vec \omega)L(\mathcal W(x, y) \leftarrow \vec \omega)d\omega dxdy
$$

其中，$(x_0, y_0)$是$C_I$中某个像素$j$的左上角，$(x_1, y_1)$则是$j$的右下角；$\mathcal W$是将$C_I$中的点转换为$\mathcal M_j$上的点的函数，$Q_e^{(j)}$则是衡量$L(\mathcal W(x, y) \leftarrow \vec \omega)$对$I_j$的重要程度的函数，从计算的角度看比$W_e^{(j)}$更为直接。

根据$C_I$的定义，$x_1 - x_0 = y_1 - y_0 = 1$，因此如果我们在$j$内均匀采样，那么概率密度函数将是值为1的常函数，这就给出了用蒙特卡路方法估算$I_j$的公式：

$$
\begin{aligned}
\hat I_j &= \frac 1 N\sum_{i=1}^N \frac{Q_e^{(j)}((x_i, y_i) \to \vec \omega_i)L(\mathcal W(x_i, y_i) \leftarrow \vec \omega_i)}{p(x_i, y_i)p(\omega_i \mid (x_i, y_i))} \\
&= \frac 1 N\sum_{i=1}^N \frac{Q_e^{(j)}((x_i, y_i) \to \vec \omega_i)L(\mathcal W(x_i, y_i) \leftarrow \vec \omega_i)}{p(\omega_i \mid (x_i, y_i))}
\end{aligned}
$$

其中$N$是采样数，$(x_i, y_i)$和$\omega_i$分别是第$i$次采样得到的$C_I$中的点和入射方向，$p(\omega_i \mid (x_i, y_i))$是在选中了$(x_i, y_i)$后选中$\omega_i$的条件概率密度。

接下来我们来尝试导出$W_e$和$Q_e$之间的关系。设$\mathcal M_j$可以被写作参数曲面$\mathcal M_j(\alpha, \beta)$，$(\alpha, \beta)$和$C_I$中像素$j$上的点间构成双射，且具有一大坨我在这里不想写的良好数学性质，那么我们可以把对$Q_e$的积分换到$\mathcal M_j$的参数空间$\Omega$上：

$$
\begin{aligned}
I_j &= \int_{x_0}^{x_1}\int_{y_0}^{y_1}\int_{\mathcal S^2}Q_e^{(j)}((x, y) \to \vec \omega)L(\mathcal W(x, y) \leftarrow \vec \omega)d\omega dxdy \\
&= \int_{\Omega}\int_{\mathcal S^2}Q_e^{(j)}((x(\alpha, \beta), y(\alpha, \beta)) \to \vec \omega)L(\mathcal M_j(\alpha, \beta) \leftarrow \vec \omega)\left|\frac{\partial(x, y)}{\partial(\alpha, \beta)}\right|d\omega d\alpha d\beta
\end{aligned}
$$

而$W_e$所满足的方程在形式上和上式非常相似：

$$
I_j = \int_{\Omega}\int_{\mathcal S^2}W_e^{(j)}(\vec x \to \vec \omega)L(\vec x \leftarrow \vec \omega)\cos\langle \vec N_x, \vec \omega\rangle \left|\frac{dA_{\mathcal M_j}}{d\alpha d\beta}\right| d\omega d\alpha d\beta
$$

于是$Q_e$和$W_e$满足：

$$
Q_e^{(j)}((x, y) \to \vec \omega)\left|\frac{\partial(x, y)}{\partial(\alpha, \beta)}\right| = W_e^{(j)}(\vec x \to \vec \omega)\cos\langle \vec N_x, \vec \omega\rangle \left|\frac{dA_{\mathcal M_j}}{d\alpha d\beta}\right|
$$

其中$\vec x$是$(x, y)$在$\mathcal M_j$上的对应点。我们把小孔摄像机的$W_e$代入上式，就能得到它的$Q_e$：

$$
\begin{aligned}
Q_e^{(j)}((x, y) \to \vec \omega) &= \mathrm{area}(\mathcal M_j)\frac{
    \delta\left(
        \vec \omega -
        \mathrm{Nor}\left(
            \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
        \right)
    \right)
}{
    \cos\langle\vec N_x, \vec\omega\rangle\mathrm{area}(\mathcal M_j)
}\cos\langle \vec N_x, \vec \omega\rangle \\
&= \delta\left(
    \vec \omega -
    \mathrm{Nor}\left(
        \boldsymbol R_{L \to W}[L, x_{S_j}, -y_{S_j}]^T
    \right)
\right)
\end{aligned}
$$

对薄凸透镜模型也有类似的结论：

$$
\begin{aligned}
Q_e^{(j)}((x, y) \to \vec \omega) &= \mathrm{area}(\mathcal M_j)\frac{L^2}{\pi r^2 \cos^4\langle \vec d, \vec \omega\rangle \mathrm{area}(\mathcal M_j)}\cos\langle \vec N_x, \vec \omega\rangle \\
&= \frac{L^2}{\pi r^2 \cos^3\langle \vec d, \vec \omega\rangle}
\end{aligned}
$$

对环境摄像机，注意到：

$$
\begin{aligned}
    x &= -\frac 1 {2\pi}w_I\theta \\
    y &= -\frac{h_I}{2} + \frac{h_I\phi}{\pi}
\end{aligned}
$$

于是：

$$
\left|\frac{\partial(x, y)}{\partial(\theta, \phi)}\right| = \left|\mathrm{Det}\left(\left[\begin{matrix}
    -w_I/(2\pi) & 0 \\
    0 & h_I/\pi
\end{matrix}\right]\right)\right| = \frac{w_Ih_I}{2\pi^2}
$$

若取摄像机球体半径为1（这与最后的结论无关），那么：

$$
\begin{aligned}
& \int_{\theta_1}^{\theta_2}\int_{\phi_1}^{\phi_2}\int_{\mathcal S^2}Q_e^{(j)}((x, y) \to \vec \omega)L(\vec x \leftarrow \vec \omega)\left|\frac{\partial(x, y)}{\partial(\theta, \phi)}\right|d\omega d\phi d\theta \\
=~& \int_{\theta_1}^{\theta_2}\int_{\phi_1}^{\phi_2}\int_{\mathcal S^2}W_e^{(j)}(\vec x \to \vec \omega)L(\vec x\leftarrow \vec \omega)\cos\langle \vec N_x, \vec\omega\rangle \cos\phi d\omega d\phi d\theta
\end{aligned}
$$

其中$\theta_1, \theta_2, \phi_1, \phi_2$是$\mathcal M_j$对应的$\theta$和$\phi$的范围边界。从上式可以解得：

$$
Q_e^{(j)}((x, y) \to \vec\omega) = \frac{2\pi^2 \delta(\vec N_x - \vec\omega)\cos\phi}{w_Ih_I\left|(\theta_2 - \theta_1)(\sin\phi_2 - \sin\phi_1)\right|}
$$

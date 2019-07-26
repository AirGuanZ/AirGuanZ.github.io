---
title: Light Tracing：算法概述和实现
key: t20190710
tags:
  - Atrc
  - Graphics
---

传统的path tracing算法从摄像机镜头向外发射射线，并不断散射、寻找光源；与之相对地，我们也可以从光源向外发射射线，并不断散射，寻找镜头。这样的算法被称作“light tracing/particle tracing”，本文将简单地介绍其原理。在这之前要声明一下，这算法基本没什么应用，最多拿来检验一下自己对光线传播理论的理解。

<!--more-->

## Importance Propagation

我们用来做path tracing的理论基础一般是所谓的rendering equation和measurement equation。非volumetric的rendering equation长这样：

$$
L(x \to \omega_o) = L_e(x \to \omega_o) + \int_{\mathcal S^2}f_s(\omega_i \to x \to \omega_o)L(x \leftarrow \omega_i)|\cos\langle n_x, \omega_i\rangle|d\omega_i
$$

其中$L(x \to \omega_o)$是场景中从$x$点向方向$\omega_o$传播的辐射亮度（radiance），$L_e$是$x$点的自发光，$\mathcal S^2$表示整个单位球面对应的立体角，$f_s$是BSDF，$n_x$是$x​$点的法线。一些基本概念在这里就不再解释了，总之这个方程告诉了我们如何求在场景中任意一点朝任意一个方向观察到的亮度。

另一个方程measurement equation对许多人来说要陌生一些，长这样：

$$
I_j = \int_{\text{sensor}}\int_{\mathcal S^2}W_e(x \to \omega)f_j(x)L(x \leftarrow \omega)d\omega dA_x
$$

其中$I_j$是像素$j$的值，$x$是摄像机传感器上的某点，$W_e(x \to \omega)$表示$x$点对来自$\omega$方向的radiance产生的响应强度（就是用来把radiance转换成颜色的映射），$f_j$则是用来重建像素$j$的filter（参见[此文](http://alvyray.com/Memos/CG/Microsoft/6_pixel.pdf)）。这里我写出的形式可能和其他一些文献给出的形式不同，比如把pbrt里的一个$\cos$项塞到$W_e$里了，这不会影响算法的正确性。不得不说measurement equation对理解path tracing来说是比较鸡肋的，即使不显式借助于它也能写出正确的渲染器。但是要反过来做light tracing，也就是要以光源为起点建立以摄像机传感器为终点的路径时，$W_e$就比较重要了。

现在我们理想化地在场景中某点$p​$放置一小束朝向$\omega​$方向的光，如果这束光恰好朝向传感器，那么它的radiance乘上传感器上对应点$x​$的$W_e(x \to -\omega)​$，就是它让$x​$点产生的直接响应。而如果这束光没有照射到传感器，那么它也会在场景中的物体间散射，并有可能在多次散射后击中传感器，让其产生间接响应。由此可见，“响应”可以定义在场景中的每个点和方向上，表示此处的光源对摄像机传感器的影响有多大。我们把“响应比”记作$W(x, p \leftarrow \omega)​$，其含义是一小束光源$L(p \to \omega)d\omega dA_p^\perp​$对传感器上的$x​$点会造成的响应与$L(p \to \omega)​$的比值。这样一来，下述积分可以求出传感器上的$x​$点对整个场景中的光源产生的响应：

$$
\int_{\text{light area}}\int_{\mathcal S^2}L_e(p \to \omega)W(x, p \leftarrow \omega)|\cos\langle n_p, \omega\rangle|d\omega dA_p
$$

而$I_j​$也就可以从measurement equation变形出来了：

$$
I_j = \int_{\text{sensor}}f_j(x)\left(\int_{\text{light area}}\int_{\mathcal S^2}L_e(p \to \omega)W(x, p \leftarrow \omega)|\cos\langle n_p, \omega\rangle|d\omega dA_p\right)dA_x
$$

早在超过20年前，学界就给“响应比”$W$起了个名字——importance function，并且指出$W, W_e$间的关系与$L, L_e$间的关系非常相似，按下面的规律传播：

$$
W(x, p \leftarrow \omega) = W_e(q \to -\omega) + \int_{\mathcal S^2}f_s^*(\omega_i \to q \to -\omega)W(x, \omega_i \to q)|\cos\langle n_q, \omega_i\rangle|d\omega_i
$$

其中$q$是从$p$出发沿着$\omega$方向与场景的第一个交点，$f_s^*$是BSDF的伴随（adjoint）形式（具体可以参考[Veach大佬的thesis](http://graphics.stanford.edu/papers/veach_thesis/)）。正如光源的$L_e$充当了所有$L$值的根本来源一样，$W_e$充当了$W$值的来源，也就是说我们可以用和路径追踪相似的方式来采样路径，只不过是以光源上的点为起点、以传感器为最终目标。

## Light Tracing

至此，light tracing算法似乎已经可以按上一节中给出的$I_j​$计算式以及$W​$的传播方程来编写了，唯一剩下的问题在于——对一条从光源出发的路径，我们无法规定它最后会落到sensor上的哪一点，也就是说我们不能像path tracing那样想在film上的哪个位置采样就在那个位置采样。

假设我们一共从以光源为起点采样了$N$条路径，分别记作$X_1, X_2, \ldots, X_N$，现以$f_j$为基础定义路径空间$\mathcal P$上的扩展滤波函数$f_j'​$，使得它适用于所有路径：

$$
f_j'(X) = \begin{cases}\begin{aligned}
	&f_j(x), &X\text{的末端为传感器上的}x\text{点} \\
	&0, &\text{otherwise}
\end{aligned}\end{cases}
$$

基于此，可以将measurement equation改写到路径空间$\mathcal P$上：

$$
I_j=\int_{\mathcal P}\mathcal T(X)f_j'(X)d\mu_X
$$

其中$\mathcal T$是路径的throughput，也就是路径起点处的$L_e$，乘上路径上所有的$\cos$项、BSDF以及最后的$W_e$等（那些没落到传感器上有效区域的路径的throughput当然就是0了）。据此，$I_j$的蒙特卡洛估计量为：

$$
\hat I_j = \frac 1 N \sum_{i=1}^N\frac{\mathcal T(X_i)f_j'(X_i)}{p(X_i)}
$$

其中$p$表示采样路径的概率密度函数。这就给出了light tracing的算法流程：

```
for i = 1 to N
	construct path Xi from light source to camera
	for j in all pixels
	    I[j] += (1 / N) * T(Xi) * fj'(Xi) / p(Xi)
```

## 实现效果

实现时有一些细节需要注意，比如：

* 一般在构造路径时都是用BSDF来采样，但在使用像小孔摄像机这样的$W_e$里含有$\delta$函数的摄像机模型时，路径中的最后一段必须在摄像机上采样，否则什么也看不见。
* 必须仔细构造像素颜色与$W_e$的关系，才能保证图像反映的是radiance，而不是其他的某种被错误放缩的量。我的考虑是：将$W_e$在sensor上归一化，并在最后的图像上乘个总像素数。

总的来说，我感觉light tracing的思想很简单，正确实现起来却比path tracing费力多了。下面看看效果：

![PICTURE]({{site.url}}/postpics/Atrc/Diary/Misc/2019_07_10_LightTracing.png)

好像还不错？等等，我们和mis path tracing对比一下，虽然有实现水平的干扰，不过结果还是很明确的：

![PICTURE]({{site.url}}/postpics/Atrc/Diary/Misc/2019_07_10_LightTracing2.png)

收敛得是真的很慢——正如许多文献所说，这算法没什么用……也就渲染caustic的时候比path tracing好些：

![PICTURE]({{site.url}}/postpics/Atrc/Diary/Misc/2019_07_10_LightTracing3.png)

注意到上图中light tracing画出来的玻璃球完全是黑的，这是因为玻璃本身的BSDF是$\delta$分布，而小孔摄像机模型的$W_e$也是$\delta$分布，采样任何一个分布都不能在二者间得到一条有效路径，也就没法画出来了。

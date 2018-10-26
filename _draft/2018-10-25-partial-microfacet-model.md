## Mircrofacet Model

### Torrance-Sparrow Framework

随着硬件性能的增长，近来微表面模型在实时渲染中也大行其道了，然而它很早就已经应用在了离线渲染中。非常奇怪的是，我很少在教材关于微表面模型的章节中看到任何数学证明——要么一言不合甩出一坨巨大的公式，要么“根据某某和某某的工作”直接给出结论，以至于我看完了全文还是不知道那些式子是怎么来的。既然如此，我就追根溯源地查找一下资料，并在本文中把这些玄学一般的公式推导一番。

微表面模型中的“微表面”意为反射光的表面由大量细小的、朝向各不相同的理想镜面构成，这些镜面是如此之小，以至于在宏观上完全看不出来这一特征。我不太懂这方面的物理/材料学，感觉不到这个说法哪里有道理，但微表面模型和现实中测量出的结果匹配得相当好，不管有无道理，拿来用是没问题的。

### Geometry Attenuation Factor

框架是有了，接下来的工作就是给出这里面的各个函数。$F_r$是介绍过的，所以我们从$G$开始。$G$的正式名称是Geometry Attenuation Factor（几何衰减因子），表达了邻近微表面间的遮蔽关系。现在我们做一个近似——忽略那些朝向有问题的微表面，只考虑一系列方向合适的、“V”字形微表面结构，分以下三种情况讨论（图片来自Blinn）：

![PICTURE]({{site.url}}/postpics/Atrc/2018_10_25_GeometryFactor.png)

注意上图只是为了说明分类，实际情形中$\Phi$（图中为$L$）和$\Theta$（图中为$E$）未必在$N_x$与$H$所确定的平面上。尽管如此，我们还是可以将它们投影到该平面上以简化推导，这并不会影响最终结果。有遮蔽的情形投影后的示意图如下，我们需要计算的是$1 - (m/l)$：

![PICTURE]({{site.url}}/postpics/Atrc/2018_10_25_GeometryFactorFirstCase.png)

其中$H$是为BRDF做出了贡献的微表面的法线。由于微表面是镜面反射，$H$其实就是$\Phi$和$\Theta$正中间的方向：

$$
H = \frac{\Phi + \Theta}{|\Phi + \Theta|}
$$

现由正弦定理：

$$
\frac m l = \frac{\sin f}{\sin b} = \frac{\sin f}{\cos e}
$$

而：

$$
\sin f = \sin(b+c) = \sin b\cos c + \cos b\sin c = \cos e\cos c + \sin e\sin e
$$

由于$c = 2d$，有：

$$
\begin{aligned}
	\cos c &= 1 - 2\sin^2d = 1 - 2\cos^2a \\
	\sin c &= 2\cos d\sin d = 2\sin a\cos a
\end{aligned}
$$

将$\sin c$和$\cos c$的表达式代入$\sin f$得：

$$
\begin{aligned}
	\sin f
	&= \cos e(1 - 2\cos^2a) + 2\sin e\cos a\sin a \\
	&= \cos e - 2\cos a(\cos e\cos a - \sin e\sin a) \\
	&= \cos e - 2\cos a\cos(e+a) \\
	&= (H\cdot E_p) - 2(N\cdot H)(N \cdot E_p)
\end{aligned}
$$

由于$E_p$是投影到$N，H$平面上的$E$，应满足：

$$
\begin{aligned}
	N \cdot E_p &= N \cdot E \\
	H \cdot E_p &= H \cdot E
\end{aligned}
$$

故：

$$
G_1 = 1 - \frac m l = \frac{2(N\cdot H)(N\cdot E)}{E \cdot H} = \frac{2(N\cdot H)(N\cdot \Theta)}{\Theta \cdot H}
$$

同理，可以推得另一种遮蔽情形下的几何因子为：

$$
G_2 = \frac{2(N\cdot H)(N\cdot \Phi)}{\Phi \cdot H}
$$

最终的$G$就是各种情形中最惨烈的那种——

$$
G = \min\left\{1, \frac{2(N\cdot H)(N\cdot \Theta)}{\Theta \cdot H}, \frac{2(N\cdot H)(N\cdot \Phi)}{\Phi \cdot H}\right\}
$$

当然，稍微动点脑子就知道这个$G$中充满了近似和不准确，因为“V”字形模型本身就缺乏根据，然而这个误差有多大我并不会分析。偏偏这个模型work得如此之好，让我都有种炼丹的的感觉了。

### Microfacet distribution function

几乎所有的微表面模型都假设微表面呈现出上面推导$G$时的“V”字形结构，这导致这些模型的几何衰减项基本都长这个样子，而$D$的形式就见仁见智了。最早的微表面模型——Torrance-Sparrow模型简单粗暴地使用用了正态分布：

$$
D(H) = e^{-(\alpha c)^2}
$$

其中$c$正比于分布标准差，$c$越大的材料显得越粗糙；$\alpha$是微表面朝向$H$和表面法线$N_x$的夹角。

（施工中……）


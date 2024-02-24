---
title: Vector displacement map生成二三事
key: t20240223
tags:
  - Graphics
---

最近遇到的一个神奇的需求，需要生成尽可能高效、准确的vector displacement map。春节在家尝试了一番，在此做个小结。

<!--more-->

## Intro

所谓vector displacement map，就是把普通displacement map的每个元素给换成一个三维向量，使得被displace的表面不仅能向法线方向移动，也能在切平面上移动，就像这样：

![WhatsVDM]({{site.url}}/postpics/VDM/what_is_vdm.gif)

可以看到，只要VDM的分辨率足够高、作为其基底的uniform grid mesh足够精细，它就可以扭出任意我们想要的形状来（当然，拓扑不能变）。

我所面临的需求是使用很低精度的VDM，尽可能精确地表达出给定的形状。

## Related Work

关键词如下，有兴趣可以去查已有的文献：

* Vector Displacement Map
* Regular Quadrilateral Mesh
* Geometry Images

## Naive Solution

来看这个mesh：

![Mesh]({{site.url}}/postpics/VDM/mesh0.png)

首先给它自动生成一套UV。严格地说，是做个固定边界的mesh parameterization。这会给mesh的每个顶点赋予一个uv。我们把uv作为“位置”，把mesh在$[0, 1]^2$上绘制出来，会得到这样的效果：

![Fix-boundary Parameterization]({{site.url}}/postpics/VDM/mesh0_param.png)

现在想象一下把uniform grid覆盖在这个$[0, 1]^2$区域上，那么这张图就告诉了我们uniform grid上的每个顶点$v$对应原始mesh的位置$p_m(v)$。将$v$在uniform grid上的原始位置记作$p_o(v)$，那么$v$对应的displacement vector就是：

$$
d(v) = p_m(v) - p_o(v)
$$

至此，我们得到了一张能把uniform grid扭出mesh形状的vdm：

![Fix-boundary Parameterization]({{site.url}}/postpics/VDM/mesh0_reconstruct.png)

## Grid Density Optimization

这张VDM似乎还不错，但经不起细看。原始mesh的上半部分在世界空间中的表面积较大，但在参数空间中的面积较小，这导致uniform grid中只有很少的顶点是用来描述这部分mesh的，进而其细节损失了不少；与此相对，mesh底座的平面部分没有任何细节，却有大量grid顶点聚集在上面，既浪费了vdm的存储，又浪费了gpu处理顶点的算力。

我们把grid精度降低到32x32，问题会更明显：

![Fix-boundary Parameterization]({{site.url}}/postpics/VDM/raw_vdm.png)

显然，grid的许多顶点被浪费在了无关紧要的地方，真正重要的形状特征却没采样到。这就是接下来需要解决的问题，我们将其划分为两个子问题：

1. 如何确定mesh的哪些区域更加重要？
2. 如何让grid在mesh的重要区域分布得更密集？

### Importance Field

第一个问题比较简单，可以很容易地设计出成吨的启发式方案，比如计算原始mesh上的曲率、计算上述vdm拟合结果和原始mesh之间的差异等。我采用的方法如下：

记三角形$f$的法线为$n_f$；对非边界边$e$，记包含$e$的两个三角形为$\{ f_1(e), f_2(e) \}$；对每个顶点$v$，记与其相接的非边界边构成的集合为$\mathcal E(v)$。那么顶点$v$的重要度为：
$$
    I(v) = \max_{e \in \mathcal E(v)}\left\{0.5 - 0.5\left(n_{f_1(e)}\cdot n_{f_2(e)}\right)\right\}
$$
说人话就是：与顶点相连的边对应的棱越是尖锐，此顶点就越是重要。

重要度计算无疑有更好的方案，但目前这个效果挺好，暂时就这样了。

### Density Optimization

我们可以从以下两个思路出发来调整网格的密度：

1. 调整mesh parameterization的结果，使得高重要度的顶点在参数空间分布得稀疏些。这在理论上是可行的，我之前使用的参数化方法仅仅是简单粗暴地使用了LSCM(least squares conformal maps)。然而我们需要在优化重要度分布的同时避免distortion，有点复杂，搁一边儿待考虑。
2. 在参数空间把uniform grid的采样位置扭一扭，使之多采样重要度高的区域。这个实现起来要简单很多，有不少思路，比如把让重要顶点在参数空间吸引附近的grid顶点，然后把grid转换为弹簧模型求解。在这里，我采用的是另一个实现方法：
   * 以每个顶点为中心，以其重要度为强度，随便选个kernel，把所有顶点splat到参数空间，得到一个importance field。
   * 把importance field归一化成一个概率分布，在固定边界点的情况下计算均匀分布到它的optimal transport，得到一个bijective transport map，那么transport的目标位置就是uniform grid的采样位置。

方案2的optimal transport虽然也很难求解，但这是个标准问题，可以直接用第三方实现，不像方案1那样需要自己修改parameterization metric。

来看看效果，第一步得到的importance field如下：

![Fix-boundary Parameterization]({{site.url}}/postpics/VDM/importance_field.png)

绿色的是mesh parameterization的结果，红色的是importance field。可以看到，石头和地面连接的部分因为变化比较锐利，形成了一圈高重要度区域；整个参数空间的正中间则对应mesh的上半部分，因顶点密集，同样形成了高重要度区域。

接下来跑个uniform distribution到importance distribution，就得到了对应的uniform grid扭法：

![Fix-boundary Parameterization]({{site.url}}/postpics/VDM/transport_map.png)

左边是transport map中grid顶点对应的位移，右边是按transport map调整grid采样位置后的结果。如我们所愿，重要度较高的区域现在有更高的grid密度。

最后，用新grid采样原始mesh，效果如下：

![Fix-boundary Parameterization]({{site.url}}/postpics/VDM/ot_vdm.png)

左边是原始版本，右边是新版本，棒极了。

## Sharp Feature Alignment

TODO

## References

TODO

---
layout: post
title: 在路径追踪中分别采样直接与非直接光照
key: t20181012
tags:
  - Graphics
---

<!--more-->

## 问题

最近开始写自己的离线渲染器[Atrc](https://github.com/AirGuanZ/Atrc)，撸个暴力的路径追踪器很是容易，代码甚至只有几行：

{% highlight c++ %}
Spectrum PathTracer::Trace(const Scene &scene, const Ray &r, uint32_t depth) const
{
    if(depth > maxDepth_)
        return SPECTRUM::RED;

    Intersection inct;
    if(!FindClosestIntersection(scene, r, &inct))
        return SPECTRUM::BLACK;

    auto bxdf = inct.entity->GetBxDF(inct);
    auto bxdfSample = bxdf->Sample(-r.direction, BXDF_ALL);
    if(!bxdfSample)
        return bxdf->EmittedRadiance(inct);

    auto newRay = Ray(inct.pos, bxdfSample->dir, 1e-5);
    AGZ_ASSERT(ValidDir(newRay));
    return bxdfSample->coef * Trace(scene, newRay, depth + 1) / SS(bxdfSample->pdf)
         + bxdf->EmittedRadiance(inct);
}
{% endhighlight %}

这里用一个最大追踪深度来截断过长的路径会导致渲染结果是有偏的，不过这一问题很容易解决（比如引入Russian Roulette策略等），不是本文要讨论的重点。从代码可以看出，路径追踪算法从摄像机镜头出发，沿着一条随机路径不断追寻光源，只有恰好击中了光源的路径才能为最终结果贡献出非零的辐射值（Radiance）。因此，光线越容易击中光源，算法收敛得越快；反之，如果光源很小或是被遮挡得很厉害，那么一条路径会以很小的概率击中光源并得到一个很亮的点，同时又以极大的概率无法击中光源从而得到黑色，这会导致画面上出现许多噪点。

![PathTracerConvergeTest]({{site.url}}/postpics/Atrc/2018_10_12_path_tracer_converge_test.png)

上图中，上方是场景形体的参考图像，中间是用淡蓝色的“天空”把场景包裹起来后用每像素100路径渲染的结果，下方则是在场景中添加一个很小的发光球体后用每像素100路径渲染的结果。可以看到，中间的场景中路径很容易击中天空这一无比巨大的光源，而下方的场景中要击中这个小球则是一个概率很小的事件，这导致同为100采样数，下方的输出噪点比中间的输出要明显得多。这并不是场景的明暗导致的，而是光源过小，很难被采样到导致的。

## 推导



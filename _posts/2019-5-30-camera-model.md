---
layout: post
title: Light Tracing初探
key: t20190530
tags:
  - Atrc
  - Graphics
---

终于把毕设做完，可以把时间花在渲染器上了。这篇文章记记Path Tracing的反面——Light Tracing。

<!--more-->

## Importance Function

设图像平面上像素$j$的重建滤波函数为$f_j$，对图像平面上的某点$p = (x_p, y_p)$和入射方向$\omega$，摄像机对radiance的响应是线性的，记作$W_e(p \to \omega)$，则像素$j$的颜色为：

$$
I_j = \left. \left(\int_{\mathrm{film}}\int_{\mathcal S^2}f_j(p)W_e(p \to \omega)L(p \leftarrow \omega)d\omega dA_p\right) \middle / \left(\int_{\mathrm{film}}f_j(p)dA_p\right) \right.
$$

设$t(x, \omega)$为从$x$向$\omega$方向发射的射线与场景的最近交点，定义$W(x \leftarrow \omega)$为$L(x \to \omega)$让摄像机在某个点产生的radiance响应比，则：



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

设图像平面上像素$j$的重建滤波函数为$f_j$，对图像平面上的某点$p = (x_p, y_p)$和入射方向$\omega$，摄像机对radiance的响应是线性的，记作$W_e^p(p \to \omega)$，则像素$j$的颜色为：

$$
I_j = \left. \left(\int_{\mathrm{film}}\int_{\mathcal S^2}f_j(p)W_e^p(p \to \omega)L(p \leftarrow \omega)d\omega dA_p\right) \middle / \left(\int_{\mathrm{film}}f_j(p)dA_p\right) \right.
$$

定义$W^p(x \leftarrow \omega)$为$L_e(x \to \omega)$让摄像机在某个点$p$产生的radiance响应比，则：

$$
I_j = \left. \left( \int_{\mathrm{film}}f_j(p) \int_S W^p(x\leftarrow \omega)L_e(x \to \omega) dA_x dA_p \right) \middle / \left(\int_{\mathrm{film}}f_j(p)dA_p \right)\right.
$$

易见：

$$
W^p(x \leftarrow \omega) = W_e^p(x \leftarrow \omega) + \int_{\mathcal S^2}f_s(\omega_i \to t(x, \omega) \to -\omega)W^p(t(x, \omega) \leftarrow \omega_i)\cos\langle N_{t(x, \omega)}, \omega_i\rangle d\omega_i
$$

其中$t(x, \omega)$是从$x$出发沿$\omega$方向的射线与场景的最近交点。

## Camera Importance Emitting

$W_e$是根据摄像机的特性和我们的需求设计出来的。

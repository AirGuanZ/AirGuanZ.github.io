---
layout: post
title: Delta函数积分换元时的放缩
key: t20181117
tags:
  - Mathematics
---

一个小坑。

<!--more-->

$$
\int_\Omega \delta(\vec\omega - \vec\omega_0) d\omega = 1~~~~(\vec\omega_0 \in \Omega)
$$

将它转换为表面积形式后：

$$
\int_\mathcal M \delta(\vec e_{xx'} - \vec e_{xx_0}) \frac{\cos\langle N_{x'}, \vec e_{x'x} \rangle}{|x - x'|^2}dA_x = \frac{\cos\langle N_{x_0}, \vec e_{x_0x} \rangle}{|x - x_0|^2}
$$

这样肯定是不行的，所以在做这个立体角-表面积转换的时候要放缩一下$\delta$：

$$
\delta(\vec\omega - \vec\omega_0) \Rightarrow \frac{|x - x_0|^2}{\cos\langle N_{x_0}, \vec e_{x_0x} \rangle}\delta(\vec e_{xx'} - \vec e_{xx_0})
$$

这个小小的问题卡了我整整一个星期……它的一般表述是这样的：

$$
\delta_\mu(x - x_0) = \frac{d\mu'}{d\mu}\delta_{\mu'}(x - x_0)
$$

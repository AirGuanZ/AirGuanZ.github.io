---
layout: post
title: Delta函数在立体角-表面积转换时的放缩
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
\int_\mathcal M \delta(\vec e_{xx'} - \vec e_{xx_0}) \frac{\cos\langle N_{x'}, e\vec e_{x'x} \rangle}{|x - x'|^2} = \frac{\cos\langle N_{x_0}, e\vec e_{x_0x} \rangle}{|x - x_0|^2}
$$

这样肯定是不行的，所以在做这个立体角-表面积转换的时候要放缩一下$\delta$：

$$
\delta(\vec\omega - \vec\omega_0) \Rightarrow \frac{|x - x_0|^2}{\cos\langle N_{x_0}, e\vec e_{x_0x} \rangle}\delta(\vec e_{xx'} - \vec e_{xx_0})
$$

所以$\delta$函数也有性质和普通函数截然不同的时候，即做积分的变量代换的时候。可惜我不懂$\delta$在数学上是怎么定义出来的。

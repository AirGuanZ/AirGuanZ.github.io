---
layout: post
title: Light Tracing：算法概述和实现
key: t20181202
tags:
  - Atrc
  - Graphics
---

传统的路径追踪算法从摄像机镜头向外发射射线，并不断散射、寻找光源；与之相对地，我们也可以从光源向外发射射线，并不断散射，寻找镜头。这样的算法被称作“Light Tracing”，本文将简单地介绍其原理。

<!--more-->

## Measurement Equation

设想我们从光源发射一条射线，它携带了一定量的radiance。当它千辛万苦到达摄像机镜头时，会对一些特定的像素颜色产生影响。那么这个影响的大小该如何给定呢？必然就需要一个用来衡量摄像机对各方向来的光的敏感程度，记作$W_e$。$W_e(x \to \Theta)$表示镜头上的$x$点对来自$\Theta$方向的radiance的响应强度。对film上的某个像素$j$，它的颜色可以被表示为：

$$
I_j = \int_{\mathcal M_j}\int_{\Omega_x}W_j(x \to \vec\omega)L(x \leftarrow \vec\omega)\cos\langle N_x, \vec\omega\rangle d\omega dA_x
$$

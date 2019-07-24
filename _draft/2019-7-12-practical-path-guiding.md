---
title: 路径追踪中的Path Guiding
key: t20190712
tags:
  - Graphics
---

主要参考《Practical Path Guiding for Efficient Light-Transport Simulation》。

<!--more-->

基于光线传播路径的全局光照算法所面临的核心问题是如何高效地采样那些携带能量多的路径，对此，一种非常直观的思路就是先粗略地采样路径空间，得到路径空间上能量分布的一个近似表示，然后使用该近似来进行更精细的采样。



## 光场表示

## 阈值选择和预算分配

## 实现效果
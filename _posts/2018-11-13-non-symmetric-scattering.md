---
layout: post
title: 非对称BRDF
key: t201811113
tags:
  - Graphics
---

绝大多数BSDF都对入射方向和出射方向对称，即：

$$
f_s(\Phi \to x \to \Theta) = f_s(\Theta \to x \to \Phi)
$$

本文讨论少数不对称的情形，它们会对诸如双向路径追踪等bidirectional算法的正确性产生影响。

<!--more-->



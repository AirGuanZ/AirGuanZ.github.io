---
layout: post
title: 真正的“光线追踪”——Light Tracing
key: t20181209
tags:
  - Atrc
  - Graphics
---

Ray tracing作为一种图形学中的常见操作，被翻译为光线追踪，这是一个非常奇怪的翻译错误。麻烦的是，它让另一个基于ray tracing的全局光照算法——light tracing在中文里没有了容身之所。本文简单介绍light tracing的。

<!--more-->

广为人知的path tracing从摄像机镜头产生射线，不断追踪其“传播”路径，直到击中光源为止；light tracing则与之相反，从光源处产生射线，不断追踪，直到击中摄像机镜头为止。

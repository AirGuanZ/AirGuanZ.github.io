---
title: Clustered Shading实现笔记
key: t20210103
tags:
  - Graphics
---

<!--more-->

## 从forward到deferred管线

古典的渲染管线（forward管线）长这样：

```cpp
for each object o:
    for each pixel p covered by o:
        p = 0
        for each light l:
            p += shade(o, l)
```

当物体遮挡多的时候，这一管线会产生大量的overdraw，也就是之前画的物体被之后画的物体遮挡住了，从而导致之前的着色计算成了无用功。如果光源数量大、着色过程复杂，overdraw将带来极为可怕的性能浪费。对此，有两种常见的解决方案：pre-depth和deferred管线。

Pre-depth就是预先跑一个深度管线，得到完整的深度图，然后再运行一次普通的forward管线，这时由于普遍存在于现代GPU上的early-z test，被挡住的物体都不会导致其对应像素着色器的运行，也就改善了overdraw问题。Pre-depth管线伪代码如下：

```cpp
set_depth_test_function(less) // default in most graphics api
for each object o:
    draw o to depth buffer
set_depth_test_function(less_or_equal)
for each pixel p covered by o:
    p = 0
    for each light l:
        p += shade(l)
```

Deferred管线则是将着色所需要的位置、法线、材质参数写入到一张（或多张）和屏幕同等大小的缓存中，这些缓存被称为G-Buffer。然后对屏幕上的每个像素，直接从G-Buffer中取出对应的位置、法线和材质参数，结合光源计算处该像素的颜色。其伪代码如下：

```cpp
for each object o:
    for each pixel p covered by o:
        p.position  = ...
        p.normal    = ...
        p.albedo    = ...
        ... // any other attributes need by shading

for each pixel p on screen:
    p.color = 0
    for each light l:
        p.color += shade(p.position, p.normal, p.albedo, ..., l)
```

此处仅根据我的个人认知总结一下这两种管线相对于古典forward管线的优缺点：

Pre-depth管线的优点：
* 改善overdraw问题
* 提前得到一张深度图，可以用作一些屏幕空间效果的输入（如SSAO）

Pre-depth管线的缺点：
* 提高了drawcall负担
* 如果采用了复杂的几何着色或细分，会浪费很多性能在几何处理上

Deferred管线的优点：
* 改善overdraw问题
* 不仅得到了深度图，还得到了法线图、材质参数图等等，有利于许多效果的实现

Deferred管线的缺点：
* G-buffer会带来较大的显存带宽占用
* 无法处理半透明物体
* 不兼容MSAA

针对这两种管线的缺点，另有一些优化方法，如通过使用现代图形API提供的并行命令构建来缓解大量drawcall的cpu负担、利用部分硬件支持的on-chip memory减小deferred管线的显存带宽问题等。因为不是本文重点，在此不多赘述。

## 对光源建立空间加速数据结构

不管是采用上文中的哪种管线，当光源数量$L$不断增长时，理论计算时间都随着$L$线性地增加，如果同一场景中出现成百上千个光源，性能就完全没法看了。

对此，注意到大多数光源都只对场景中的一部分区域有显著影响，如果某个光源对某个像素的影响极小，那么在对该像素进行着色时，直接将此光源忽略也是可以接受的。据此，不妨为每个光源划定一片受它影响的空间范围，从而可以对全体光源建立一个空间加速数据结构，进行像素着色时，首先查询该结构，然后仅使用查询结果中的光源进行计算。也就是把原来的：

```cpp
// to shade pixel p:
for each light l:
    p += shade(l)
```

变成了：

```cpp
// to shade pixel p:
ls = query_lights(p)
for each light l in ls:
    p += shade(l)
```

在实践中，ls的大小很可能是所有光源数量的十分之一乃至百分之一，对性能的提升极为显著。
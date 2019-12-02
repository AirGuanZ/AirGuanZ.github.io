---
title: Shimmering in Shadow Mapping
key: t20191201
tags:
  - Graphics
---

最近在自己的玩具项目里实现了级联阴影贴图（Cascade Shadow Mapping, CSM），这里记录一下解决其中的一些artifacts的过程。

<!--more-->

## CSM简介

在产生方向光的阴影时，用单张阴影贴图（Shadow Map，SM）覆盖一个较大的场景会带来以下问题：

1. 由于场景较大，SM分辨率稍低就会产生严重的锯齿
2. 在离摄像机远的地方，锯齿现象并不是那么显眼，SM的高分辨率被浪费了

一种直观的解决方法是使用多张SM，每张负责场景中不同的区域。在离摄像机近的地方使用高分辨率的贴图覆盖较小的区域，远的地方则使用低分辨率的贴图覆盖较大的区域，这就是CSM的基本思想了。

使用$N$张SM，也就意味着我们需要求出$N$个不同的阴影投影矩阵，且分别使用这些投影矩阵将场景渲染$N$次。我们用与摄像机的距离将camera frustum划分成$N$段，离摄像机越远，划分距离越长。CSM的基本原理并非本文重点，所以就用下面的伪代码一笔带过吧：

```cpp
// shadow view matrix将world space中的点变换到shadow view space
// 该矩阵的值由光的方向确定
Mat4 shadowView = ...;

// 计算从camera view space到shadow view space的变换矩阵
Mat4 cameraViewToShadowView = camera.GetViewMatrix().inverse() * shadowView; 

float yOverD = 0.5f * camera.GetFOVY();
float wOverH = camera.GetAspectRatio();

float near = camera.GetNearDistance();
float far  = FAR_DISTANCE_OF_FIRST_SHADOW_MAP;

for(int smIdx = 0; smIdx < N; ++smIdx)
{
    float nearY = yOverD * near;
    float nearX = wOverH * nearY;
    float farY  = yOverD * far;
    float farX  = wOverH * farY;

    // 求出camera view space中被分割的frustum的八个端点

    Vec3 frustumPoints[8] =
    {
        { +nearX, +nearY, near },
        { +nearX, -nearY, near },
        { -nearX, +nearY, near },
        { -nearX, -nearY, near },
        { +nearX, +nearY, far  },
        { +nearX, -nearY, far  },
        { -nearX, +nearY, far  },
        { -nearX, -nearY, far  },
    };

    // 将frustum的八个端点变换到shadow view space

    for(auto &p : frustumPoints)
    {
        p = cameraViewToShadowView.transformPoint(p);
    }

    // 求出camera frustum在shadow view space中的包围盒

    Vec3 low  = frustumPoints[0];
    Vec3 high = frustumPoints[0];
    for(int i = 1; i < 8; ++i)
    {
        low  = Min(low,  frustumPoints[i]);
        high = Max(high, frustumPoints[i]);
    }

    // 用该包围盒构建第smIdx级SM的投影矩阵
    // 注意到投影矩阵中的minZ没有使用low.z，而是使用了一个预设值
    // 否则会导致frustum之外、离光源更近的物体所产生的阴影丢失

    Mat4 shadowProj = Mat4::Orthographic(low.x, high.x, low.y, high.y, SHADOW_PROJ_MIN_Z, high.z);

    // 用shadowView和shadowProj渲染第simIdx级SM
    // ...

    RenderShadow(shadowView * shadowProj);

    // 计算下一级SM对应的frustum的距离
    // 这里采用的策略是第i+1级覆盖的距离是第i级距离的两倍，应针对具体场景进行调整

    float distance = far - near;
    near = far;
    far += 2 * distance;
}
```

在使用CSM渲染场景时，需要在像素着色器中用像素与摄像机的距离来选择采样哪一级SM（这里使用的是HLSL）：

```cpp
float sampleCascadeShadowMap(PSInput input)
{
    for(int smIdx = 0; smIdx < SHADOW_MAP_COUNT; ++smIdx)
    {
        if(input.clipSpaceZ < ClipSpaceLimits[smIdx])
        {
            return sampleShadowMap(smIdx);
        }
    }
    return 1;
}
```

其中，`input.clipSpaceZ`是该像素在裁剪空间中的z坐标，用于和每一级SM的远截面在裁剪空间中的z坐标比较，来确定是否采样该级SM。

下图是使用三级CSM实现的阴影，图中我用不同的亮度条带来标记采样了哪一级SM：

![PICTURE]({{site.url}}/postpics/basic-csm.png)

## 问题

Surface Acne和Peter Panning是所有Shadow Mapping类技术都面对的问题，一般来说就是仔细调节Depth Bias和Slope值，没什么好说的。我所面对的主要问题是：由于阴影区域是跟随摄像机移动的，而SM的分辨率又不是很高，采样不足，于是摄像机一移动或是旋转，阴影边缘的锯齿就会疯狂闪烁，就像这样：

![PICTURE]({{site.url}}/postpics/shadow-shimmering-0.gif)

在使用了PCF采样等模糊SM的方法后，静态的阴影锯齿变得不那么扎眼，但移动的锯齿就不一样了，时刻向观察者提醒自己的存在。

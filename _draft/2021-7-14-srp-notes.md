---
title: Unity SRP学习笔记
key: t20210714
tags:
  - Unity
  - Graphics
---

最近在学习Unity Scriptable Render Pipeline，没啥原理，但需要死记硬背的东西比较多，姑且在这写个备忘。

<!--more-->

### 创建Custom Render Pipeline

1. 添加`class CRPAsset`，继承`RenderPipelineAsset`
2. 添加`class CRP`，继承`RenderPipeline`

`CRPAsset`在`CreatePipeline`中创建一个`CRP`实例。渲染管线的主要逻辑在`CRP`中实现。

### 渲染某个摄像机的基本流程

1. `context.SetupCameraProperties(camera)`
2. 记录一系列渲染操作。某些操作只能通过`CommandBuffer`先记录好，然后通过提交给`context`，提交之后最好即时调用`cmds.Clear()`
3. `context.Submit()`

### 基本渲染流程

1. 从摄像机取得culling参数，其中包含frustum和layer mask等信息
2. `context.Cull`得到`CullingResult`
3. 准备`SortingSettings`，`DrawingSettings`和`FilteringSettings`，和裁剪结果一起丢给`context.DrawRenderers`

`SortingSettings`主要描述渲染顺序，比如对透明物体经常采用从后往前什么的；`DrawingSettings`包括了`SortingSettings`，`Shader LightMode`以及instancing等设置；`FilteringSettings`主要是基于layer和queue筛选要渲染哪些物体。

### Transforms in Shader

因为gfx兼容性、统一不同使用情形（如开启/关闭instancing）等原因，Unity传输物体几何变换数据的方式不怎么透明，最好使用官方package定义的一些宏来访问这些常用的几何变换数据：

```cpp
// for common types like real4
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"

#define UNITY_MATRIX_M   unity_ObjectToWorld
#define UNITY_MATRIX_I_M unity_WorldToObject
#define UNITY_MATRIX_V   unity_MatrixV
#define UNITY_MATRIX_VP  unity_MatrixVP
#define UNITY_MATRIX_P   glstate_matrix_projection

float4x4 unity_ObjectToWorld;
float4x4 unity_WorldToObject;
real4    unity_WorldTransformParams;
float4x4 unity_MatrixVP;
float4x4 unity_MatrixV;
float4x4 glstate_matrix_projection;

// for common geometry transforms like TransformObjectToWorld
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"
```

### 材质参数

可以在shaderlab里通过固定格式直接定义使用该shader的材质有什么参数。基本格式：

```
Properties {
    _PropertyName("PropertyNameString", PropertyType) = PropertyDefaultValue
}
```

比如：

```
_BaseColor("Color", Color) = (1.0, 1.0, 1.0, 1.0)
```

### SRP Batcher

![](https://blog-api.unity.com/sites/default/files/2019/02/image5-3.png)

古老的Unity Render Loop中，每当我们需要使用一种新的材质来画下一个物体时，Unity会整理材质需要的数据，填充对应的cbuffer，然后绑定给shader用。SRP Batching的大概思路就是把这些数据都预先在GPU Buffer里面准备好，只有物体发生了变化才需要去重新准备，否则就直接把之前准备好的cbuffer拿来用。至于几何数据，仍然需要由专门的代码去不断填充cbuffer。

启动SRP Batcher：

```csharp
GraphicsSettings.useScriptableRenderPipelineBatching = true;
```

要让一个物体可以被SRP Batcher处理，有以下要求：

* 是普通的mesh，不是粒子或者skinned mesh之类的
* shader的builtin properties都统一存在一个叫UnityPerDraw的cbuffer里
* 材质参数都统一存在一个叫UnityPerMaterial的cbuffer里
* 没有用上MaterialPropertyBlock，这玩意儿可以乱改材质属性

PerDraw的数据不能乱定义，应该按照下表以“block feature”为单位选择性地往里面放，对某个block feature中的数据，顺序也不能乱：

![img](https://blog-api.unity.com/sites/default/files/2019/02/Screen-Shot-2019-02-27-at-3.50.52-PM.png?imwidth=2048&)

此外，对任意一个feature block，如果把其中的某个var定义成了half，这个block里的其他可定义为half的var也都要对应地定义为half。

### MaterialPropertyBlock

用于逐object地设置材质属性：

```csharp
int baseColorId = Shader.PropertyToID("_BaseColor");\
block.SetColor(baseColorId, baseColor);
GetComponent<Renderer>().SetPropertyBlock(block);
```

### GPU Instancing

* 通过`#pragma multi_compile_instancing`启用instancing版本的shader变体
* 引入`Packages/com.unity.render-pipelines.core/ShaderLibrary/UnityInstancing.hlsl`
* 在`VertexInput`中加入`UNITY_VERTEX_INPUT_INSTANCE_ID`
* 在`Vertex Shader`中加入`UNITY_SETUP_INSTANCE_ID(input);`和`UNITY_TRANSFER_INSTANCE_ID(input, output);`
* 使用如下格式定义instancing相关的cbuffer：
  ```hlsl
    UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
        UNITY_DEFINE_INSTANCED_PROP(float4, _BaseColor)
    UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)
  ```
* 使用`UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor)`访问instancing相关的属性

也可以通过`MaterialPropertyBlock`批量设置材质属性，然后用`Graphics.DrawMeshInstanced`直接画一堆物体，而不必依赖Unity的Game Object自动去合成instancing调用。

### Clip Space

Unity的Proj矩阵生成的ClipSpace的Z范围是\[-1, +1\]



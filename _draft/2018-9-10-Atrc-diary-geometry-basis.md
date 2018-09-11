---
layout: post
title: Atrc日志（一）几何对象接口
key: t20180910
tags:
  - Atrc
  - Graphics
---

自己编写一个效果能看的离线渲染器的想法在两年前就已经萌发了，这段时间开始动手，感觉在PBRT和大量参考实现的帮助下也不算太困难。

项目代码位于[Atrc](https://github.com/AirGuanZ/Atrc)，目前还处于非常早期的阶段。

<!--more-->

首先是一些基本的数学概念（Transform，Ray，Intersection等）的编程实现，这部分代码我是基于(Utils/Math)[https://github.com/AirGuanZ/Utils/blob/master/Src/Utils/Math.h]完成的。

## Ray

一个Ray对象表示的是一条参数化的射线$\bm p = \bm o + t\bm d~(t \ge 0)$，因此其存储非常简单：

{% highlight c++ linenos %}
class Ray
{
public:

    Vec3r origin;
    Vec3r direction;

    Ray() = default;

    Ray(const Vec3r &ori, const Vec3r &dir)
        : origin(ori), direction(dir)
    {

    }

    Vec3r At(Real t) const
    {
        return origin + t * direction;
    }

    void Normalize()
    {
        direction = AGZ::Math::Normalize(direction);
    }
};
{% endhighlight %}

代码中的`Vec3r`表示的是`Vec<Real>`，是以实数（Real，简写作r）为分量类型的三维向量。`At`可以求出给定$t$对应的射线上的点，`Normalize`则用于将方向向量$\bm d$单位化。

## Transform

Transform用来表示三维空间中诸如平移、旋转、缩放等几何变换。常用的变换表示形式是四阶方阵，它已被Utils/Math实现。因此对变换的封装也就变得容易起来：

{% highlight c++ linenos %}
class Transform
{
    Mat4r trans_;
    Mat4r invTrans_;

public:

    Transform();
    explicit Transform(const Mat4r &trans);
    Transform(const Mat4r &trans, const Mat4r &invTrans);
    explicit Transform(const Mat4r *trans, const Mat4r *invTrans = nullptr);

    static const Transform &StaticIdentity();

    Transform operator*(const Transform &rhs) const;

    Transform Inverse() const;

    Vec3r ApplyToPoint(const Vec3r &p) const;
    Vec3r ApplyToVector(const Vec3r &v) const;
    Vec3r ApplyToNormal(const Vec3r &n) const;

    Vec3r ApplyInverseToPoint(const Vec3r &p) const;
    Vec3r ApplyInverseToVector(const Vec3r &v) const;
    Vec3r ApplyInverseToNormal(const Vec3r &n) const;

    Ray ApplyToRay(const Ray &ray) const;
    Ray ApplyInverseToRay(const Ray &ray) const;

    SurfaceLocal ApplyToSurfaceLocal(const SurfaceLocal &sl) const;
};
{% endhighlight %}

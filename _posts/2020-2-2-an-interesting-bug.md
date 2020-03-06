---
title: C++中的一个由析构顺序引发的bug
key: t20200202
tags:
  - C/C++
---

今天刚解决的一个小bug，觉得很有意思，记录一下。

<!--more-->

本文有严重事实错误，待修改

==========================================================================

在C++中的构造/析构函数里调用虚函数通常不是个好主意。考虑如下代码：

```cpp
class A
{
public:

    ~A()
    {
        cout << "A destructor" << endl;
    }
};

class Base
{
public:

    ~Base()
    {
        cout << "Base destructor" << endl;
    }
};

class Derived : public Base
{
    A a;

public:

    ~Derived()
    {
        cout << "Derived destructor" << endl;
    }
};

int main()
{
    Derived derived;
}
```

运行这个程序得到的输出是：

```
Derived destructor
A destructor
Base destructor
```

可以看到，如果在`Base::~Base()`里调用了虚函数，此时`Derived::A`已经析构掉了，而调用的虚函数是有可能访问到`Derived::A`的，这会引发未定义行为。

这个坑的原理非常简单，熟悉构造/析构函数执行顺序的人会觉得：这有什么有趣的，非常trivial的bug而已。的确，直接写出这个bug显得有点蠢，但有时这一问题会被一堆乱七八糟的其他逻辑藏起来。我今天遇到的这个bug出现在需要多线程的离线渲染里，被隐藏在好几层继承以及多线程之后，原理是这样的：

```cpp
class Base
{
    // ... : some other members

    atomic<bool> stop_ = false;

    thread worker_;

protected:

    virtual void foo() = 0;

public:

    virtual ~Base()
    {
        stop();
    }

    void start()
    {
        stop_ = false;
        worker_ = thread([this]
        {
            // do something

            while(!stop_)
                foo();

            // do something
        });
    }

    void stop()
    {
        stop_ = true;
        if(worker_.joinable())
            worker_.join();
    }
};

class Derived : public Base
{
    // ... : some other members

protected:

    void foo() override
    {
        // do something
    }
};

int main()
{
    Derived derived;
    derived.start();
}
```

如上述代码，对虚函数的调用被隐藏在由`Base`创建的另一个线程中。`Base::~Base()`对`worker_`执行了`join`，但此时`Derived::~Derived()`已经被调用了；也就是说，在“`Derived::~Derived()`被调用”到“`Base::~Base()`完成对`worker_`的`join`”之间的这段时间里，`worker_`对应的线程可能会调用虚函数`foo`，进而访问`Derived`实例中已经被销毁的各种设施，引发未定义行为。

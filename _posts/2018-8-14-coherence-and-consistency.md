---
layout: post
title: 读书笔记：A Primer on Memory Consistency and Cache Coherence
key: 20180814
tags:
  - 体系结构
---

一直对存储一致性方面的东西一知半解，拖下去也不是办法，索性系统地学一学。此为读书笔记。

<!--more-->

## 引言

多个处理器核心读写同一地址空间，这就是现在主流的shared memory system。由于在多核心情境下，有多种不同的系统行为都被视为是正确的，因此shared memory system的正确性定义也就变得困难起来，特别是硬件设计者常常期望这一定义宽松一些，使得他们能做更多的优化。

Coherence和consistency是对shared memory correctness的人为划分，我暂且把它理解为通常所说的缓存一致性和内存一致性。

Coherence的正确性很简单：cache对一切软件透明。我们除了能通过一些性能测试来估计cache的性质（比如CSAPP中那个著名的存储器山）外，无法从语义的层面获知关于cache的任何信息，只需要当它不存在就行了。

Consistency correctness则会真真切切地影响软件的行为，它决定了程序所观测到的许多内存操作的结果。我曾经在看[folly](github.com/facebook/folly)源代码时看到以下代码：

{% highlight c++ %}
static size_t refs(Char * p) {
  return fromData(p)->refCount_.load(std::memory_order_acquire);
}

static void incrementRefs(Char * p) {
  fromData(p)->refCount_.fetch_add(1, std::memory_order_acq_rel);
}

static void decrementRefs(Char * p) {
  auto const dis = fromData(p);
  size_t oldcnt = dis->refCount_.fetch_sub(1, std::memory_order_acq_rel);
  FBSTRING_ASSERT(oldcnt > 0);
  if (oldcnt == 1) {
    free(dis);
  }
}
{% endhighlight %}

显然这是某个跨线程引用计数类的成员函数，我过去都是直接拿`std::atomic<int>`怼的。可是这个`std::memory_order_acquire`还有下面的`std::memory_order_acq_rel`是些啥玩意儿？我隐约记得自己在看前半本《C++ concurrency in action》的时候遇到过它们，当时就没怎么看懂。事实上，这一套东西就是内存一致性模型在C++中的体现，编译器会根据用户所指定使用的模式，结合目标平台所使用的一致性模型，生成尽可能优化的、符合用户期望行为的代码。当然，如果心里没数（像我这样），老老实实用锁也是可以的。

## Basic Coherence

设想某芯片上有两个核心，它们各自拥有独享的L1缓存，共享L2缓存和主存。现在核心A和核心B都读取某个内存地址m处的值，都得到了结果42。在之后的某个时候，核心A把这个值改写成了43，但是此时核心B的L1缓存中的m处值依然是42，形成了cache incoherence。要避免这样的尴尬情境，就要定义一套cache coherence协议来保证cache如我们希望地那样工作。

这里且不论如何设计和实现cache coherence协议，我们先把自己想要什么给说清楚，即准确地定义cache coherence是什么。

最简单，也最直观的一个约束被称为single-writer-multiple-reader（SWMR）invariant。顾名思义，就是说在任意时刻只能有一个写者和多个读者。在SWMR约束中，时间轴被分为一系列相邻的时间段，每个时间段中要么只有一个核心在写，要么有0个或多个核心在读。此外，任何时间段内某核心读出的值都必须等于在此之前最后一个被写入被读取地址处的值，这被称作data value invariant。

需要注意的是，在理论上cache coherence可以在任意粒度上定义，但实践中它往往是以一定大小的cache block为单位实现的。此时，SWMR保证对任意一个内存块，同一时刻至多有一个写者或多个读者。

## Sequential Consistency

Coherence和consistency在理论上是可以没什么关系的，即存在这么一个系统，它满足特定的consistency，却不满足cache coherence。但在现实中，cache coherence基本都是实现了的，因此在下面对consistency的讨论中，若非指明，均假设系统满足coherence。

考虑下面的程序：

```
Core 1:
        data = VALUE
        flag = 1
Core 2:
    L:  load r = flag
        if flag == 0
            goto L
        load d = data
```

假设一开始`data`未初始化，`flag`为0。很不幸，在许多平台上这段代码都是buggy的——硬件对内存操作的重排序可能会导致Core 2看到flag被设置为1了，却发现data仍然处于为初始化的状态。事实上，在现代处理器上，memory access reordering是非常常见（大概人手一个吧……）的优化，可以被粗略地分为以下三类：

1. Store-store reordering：如果某个core使用non-FIFO write buffer，那么两个store操作就可能被重排。比如第一个store操作cache miss了，第二个命中了或者和更早的一个store合并了，第二个就能比第一个store更早地完成。即使core没有改变指令的执行顺序，这样的reordering也可能发生。
2. Load-load reordering：主要由instruction reordering造成。
3. Load-store and store-load reordering：即使按program order老老实实执行，最常见的FIFO write buffer也可能导致这种重排。

Sequential consistency（SC）：在每个core看来，memory order都符合program order。若某个core的操作A在program order层面先于操作B，就记$A <_p B$；若某个内存操作A在memory order层面先于B，则记作$A <_m B$。设a和b分别是两个内存地址（可能相同），Sequence consistency可以被以下几条规则表述：

$$
\begin{aligned}
& L(a) <_p L(b) \Rightarrow L(b) <_m L(b) \\
& L(a) <_p S(b) \Rightarrow L(b) <_m S(b) \\
& S(a) <_p L(b) \Rightarrow S(a) <_m L(b) \\
& S(a) <_p S(b) \Rightarrow S(a) <_m S(b) \\
& \text{ValueOf}(L(a)) = \text{ValueOf}(\max_{<_m}\{ S(a) \mid S(a) <_m L(a) \})
\end{aligned}
$$

除此之外，atomic read-modify-write（RMW）操作要求在它的load和store之间不能有任何别的内存操作插进来（即使操作地址不同）。这就是SC模型的全部约束了。

当然，一个好的SC实现还应该避免starvation和尽量保证fairness，不过这不在模型的讨论范围内。

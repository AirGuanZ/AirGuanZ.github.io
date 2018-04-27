---
layout: post
title: OS kernel实现日记（二）
key: 20180427
tags:
  - OS
---
<style>
img{
    width: 60%;
    padding-left: 20%;
}
</style>

多级位图实现物理内存管理

<!--more-->

## 多级位图

除了一开始内核已经占用的物理内存外，剩余物理内存均由一个专门的物理内存分配器管理，它提供物理页的分配和释放接口。TinyOS要求一个合法的物理内存分配器实现以下函数：

{% highlight c %}
/*
    分配一个物理页，返回值为物理地址
    resident为false表示非常驻，反之表示常驻内存（不会被换出）
*/
uint32_t alloc_phy_page(bool resident);

/*
    释放一个物理页
    若页面不在物理页管理器管理范围内，或本来就没被占用，则返回false
*/
void free_phy_page(uint32_t page_phy_addr);
{% endhighlight %}

分配器在初始化时，拥有某个地址区间内的全部物理页框，每当`alloc_ker_page`被调用，就将其中某个页的物理地址返回，并将其标记为已使用；释放时则将其标为可用。我早有耳闻的是Linux采用Buddy System管理物理内存，但一直不明了这么做有什么好处，所以这次并未模仿其设计。

TinyOS采用位图记录物理页框的使用情况，一个位为1表明其对应的物理页是空闲的。设想一开始的空闲页框下标范围为`[B, E)`，则需要`ceil((E-B)/32)`个`uint32_t`位图记录其可用性，这一堆位图称为L3位图。然而，分配页框时线性查找这些位图中的可用位可能会极为低效，因此在L3基础上增加一层位图，对每个L3位图均用一个位来记录其是否还有空闲空间，这一层位图需要`ceil(ceil((E-B)/32)/32)`个`uint32_t`。以此类推，可以定义出L1位图和L0位图。容易证明，即使可用物理内存达到32位地址线的极限——4GB，也只需要1个`uint32_t`类型的L0位图。多级位图示意图如下：

![]({{site.url}}/postpics/multi-level-bitmap.png){:align=center}

TinyOS的位图索引是以`uint32_t`为单位进行的，这意味着每一曾查询都需要找到32个位中的某个为1的位（为1表示可用），而线性扫描所有位来完成这件事可能浪费大量时间。一种软件解决方案是二分地在一个`uint32_t`中查找为1的位，硬件解决方案则是使用`bsr/bsf`这样的特殊指令来获得目标位的位置。由于x86上的指令周期在不同的实现上极不稳定，TinyOS暂且采用第二种方案（至少在我的i74700MQ上，这两条指令相当高效），同时保留使用第一种方案的可能。

多级位图中Li和Lj (j = i + 1)级间的关系为：

```c
Li[n] = ((Lj[32 * n + 0]  != 0) << 0) |
        ((Lj[32 * n + 1]  != 0) << 1) |
        ......
        ((Lj[32 * n + 31] != 0) << 31)
```

综上所述，TinyOS用来管理物理内存的数据结构定义为：

{% highlight c linenos %}
struct mem_page_pool
{
    // mem_page_pool是个变长结构体，所以大小记一下
    size_t struct_size;

    // 应满足begin < end，begin和end值的单位为4K
    // 该内存池的范围为[begin, end)
    // pool_pages = end - begin
    size_t begin, end;
    size_t pool_pages, unused_pages; //总页数和当前空余页数

    // 一个L0位图就足以覆盖4GB内存
    // 1表示尚可用，0表示已被占用
    uint32_t bitmap_L0;

    // 应满足：
    //    bitmap_count_L1 = ceil(bitmap_count_L2 / 32)
    //    bitmap_count_L2 = ceil(bitmap_count_L3 / 32)
    //    bitmap_count_L3 = ceil((end - begin) / 32)
    size_t bitmap_count_L1;
    size_t bitmap_count_L2;
    size_t bitmap_count_L3;

    uint32_t *bitmap_L1;
    uint32_t *bitmap_L2;
    uint32_t *bitmap_L3;

    // 页面是否应常驻内存所构成的位图
    // 1表示常驻，0表示可换出
    uint32_t *bitmap_resident;

    // gcc扩展：零长数组
    uint32_t bitmap_data[0];
};

/* 全局物理内存池 */
static struct mem_page_pool *phy_mem_page_pool;
{% endhighlight %}

当然，位图本身也要占用一定的空间，TinyOS将它放置在物理内存中$\text{0x00300000} \sim \text{0x003fffff}$这段内核保留区域的开头处。

## 分配和释放物理页

基于上述的多级位图结构，空闲物理页的查找可按以下步骤完成：

1. 在唯一的L0位图中查找一个有空闲的L1位图，设找到的下标为`l1`
2. 在`L1[l1]`中查找一个有空闲的L2位图，设找到的下标为`l2`
3. 在`L2[32 * l1 + l2]`中查找一个有空闲的L3位图，设找到的下标为`l3`
4. 在`L3[32 * 32 * l1 + 32 * l2 + l3]`中查找一个空闲位，设找到的下标为`l`
5. 空闲物理页的地址为
   ```c
   phy_page_addr = page_size * (begin + 32 * 32 * 32 * l1
                                      + 32 * 32 * l2
                                      + 32 * l3
                                      + l);
   ```

找到空闲物理页后，需要将其对应的L3位图中的位置0,并更新可能因此而改变的L0~L2位图。综上所述，物理页分配的C语言实现如下：

{% highlight c linenos %}
uint32_t alloc_phy_page(bool resident)
{
    // 现在是没了物理页直接挂掉
    // 以后有了调页机制和swap分区，再考虑换出到磁盘
    if(phy_mem_page_pool->unused_pages == 0)
        FATAL_ERROR("system out of memory");

    // 找到一个空闲物理页
    uint32_t idxL1          = _find_lowest_nonzero_bit(phy_mem_page_pool->bitmap_L0);
    uint32_t localL2        = _find_lowest_nonzero_bit(phy_mem_page_pool->bitmap_L1[idxL1]);
    uint32_t idxL2          = (idxL1 << 5) + localL2;
    uint32_t localL3        = _find_lowest_nonzero_bit(phy_mem_page_pool->bitmap_L2[idxL2]);
    uint32_t idxL3          = (idxL2 << 5) + localL3;
    uint32_t local_page_idx = _find_lowest_nonzero_bit(phy_mem_page_pool->bitmap_L3[idxL3]);

    // 设置L3位图中对应的标记位
    clr_local_bit(&phy_mem_page_pool->bitmap_L3[idxL3], local_page_idx);
    if(resident)
        set_local_bit(&phy_mem_page_pool->bitmap_resident[idxL3], local_page_idx);
    else
        clr_local_bit(&phy_mem_page_pool->bitmap_resident[idxL3], local_page_idx);

    // 更新L2、L1以及L0位图
    if(phy_mem_page_pool->bitmap_L3[idxL3] == 0)
    {
        clr_local_bit(&phy_mem_page_pool->bitmap_L2[idxL2], localL3);
        if(phy_mem_page_pool->bitmap_L2[idxL2] == 0)
        {
            clr_local_bit(&phy_mem_page_pool->bitmap_L1[idxL1], localL2);
            if(phy_mem_page_pool->bitmap_L1[idxL1] == 0)
                clr_local_bit(&phy_mem_page_pool->bitmap_L0, idxL1);
        }
    }

    --phy_mem_page_pool->unused_pages;
    
    return (phy_mem_page_pool->begin + (idxL3 << 5) + local_page_idx) << 12;
}
{% endhighlight %}

类似地，释放物理页只需要将其L3位图中的对应位设置为1（表示可用），并更新L0~L2位图即可：

{% highlight c linenos %}
void free_phy_page(uint32_t page_phy_addr)
{
    size_t page_idx = (page_phy_addr >> 12) - phy_mem_page_pool->begin;
    ASSERT(page_idx < phy_mem_page_pool->end,
           "invalid page addr (page_idx >= phy_mem_page_pool->end)");

    uint32_t idxL3          = page_idx >> 5;
    uint32_t local_page_idx = page_idx & 0x1f;
    uint32_t idxL2          = idxL3 >> 5;
    uint32_t idxL1          = idxL2 >> 5;

    // 尝试释放一个没被占用的物理页，fatal error之
    if(get_local_bit(phy_mem_page_pool->bitmap_L3[idxL3], local_page_idx))
        FATAL_ERROR("freeing unused phy mem page");

    // 设置L3位图中的标记位
    set_local_bit(&phy_mem_page_pool->bitmap_L3[idxL3], local_page_idx);
    set_local_bit(&phy_mem_page_pool->bitmap_L2[idxL2], idxL3 & 0x1f);
    set_local_bit(&phy_mem_page_pool->bitmap_L1[idxL1], idxL2 & 0x1f);
    set_local_bit(&phy_mem_page_pool->bitmap_L0, idxL1);

    //resident位图无需设置，因未被使用的物理页的resident位不具有任何含义

    ++phy_mem_page_pool->unused_pages;
}
{% endhighlight %}

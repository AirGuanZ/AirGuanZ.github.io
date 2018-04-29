---
layout: post
title: OS kernel实现日记
key: 20180423
tags:
  - OS
---

一只菜鸡的操作系统之旅

<!--more-->

## 引言

这个学期有操作系统的专业课，我很早以前已经学过一点皮毛，但几乎所有的教材都对许多细节语焉不详，读之犹如隔靴瘙痒，很不痛快。所以干脆拉了两个同学，试着自己实现一个小型操作系统。虽说找了队友，热心于此的还是只有我一个人，导致最后我疯狂commit了95%以上的代码……以前在知乎上看到有人说一个学得还不错的本科生应该有做出玩具操作系统的能力，当时还觉得这人信口开河，装得过分了，如今自己一试，的确没有当初想象的那么困难。

我把这个小系统称为“TinyOS”，截至到目前为止，TinyOS已经实现了以下特性：

- 分页式内存管理
- 内核线程调度
- 内核进程和用户进程
- 系统调用
- 信号量等基本互斥与同步机制
- 进程消息队列
- 键盘驱动

接下来的计划则是：

- 硬盘驱动和文件系统
- ELF装载器
- 终端

亲手编写操作系统这种在计算机体系中地位崇高的软件，并且只是为了学习和娱乐（写出来又没人用），这是何等浪漫的事情，不写点东西记下此过程中的酸甜苦辣和思考过程就太可惜了，故有此系列文章。

已经过去的开发历程我已经无法详述了，因此本篇所谓的“日记”只包含一些技术上的内容。

## 开发环境和项目构建

TinyOS的开发主要借助于x86虚拟机bochs，我为它编（chao）写（xi）了一个方便的makefile，可以输入`make bochs`直接在构建系统并bochs中运行它。makefile大概流程是：

1. 寻找内核源代码目录下所有的.c文件，调用gcc -MM为每个文件生成依赖文件并包含到makefile中，然后将其编译为一大堆对应的.o。
2. 寻找内核源代码目录下所有的.s文件，调用nasm将其汇编成对应的.bin。
3. 将内核编译出的.o和.bin文件链接为kernel.bootbin，该文件是ELF32格式的。
4. 将boot目录下的mbr.s和bootloader.s分别编译为mbr.bootbin和bootloader.bootbin。这两个输出文件是纯二进制文件，不带任何文件头，也不需要链接。
5. 用dd命令将mbr.s、bootloader.bootbin和kernel.bootbin分别写入磁盘映像文件的指定扇区中。

## 系统启动

x86计算机开机后，首先运行的程序是由硬件厂商编写的BIOS，轮不到我插手。BIOS在完成了基本的初始化和硬件检查后，就检查磁盘的首个扇区，若它的末尾两个字节是预先规定的magic number，BIOS就认为该扇区包含了主引导记录（Main Boot Record，MBR），会将其加载到内存的0x7c00处并跳转到该地址执行。

从MBR开始，这天下就轮到我做主了。然而MBR的容量实在有限（不到一个扇区），这点代码莫说让操作系统大展身手，就是用来加载和初始化操作系统的程序——bootloader都施展不开。因此，MBR要做的事情只有一件，就是将bootloader从磁盘加载到内存中，然后将权利移交给bootloader。

Bootloader是真正开疆拓土的程序，它有一个长长的任务列表：

1. 探测物理内存容量，以后操作系统的内存管理机制会用到
2. 打开A20地址线，解放1M以上的物理内存空间
3. 填充全局描述符表，即进行初始的“分段”
4. 由所处的实模式进入保护模式
5. 启用分页机制，创造第一个虚拟地址空间
6. 装载系统内核，迎接kernel的降临

我不打算在这里陈列无穷多的技术细节，因此只大概阐述每个步骤的做法。

1. BIOS为继任者准备了一些可用的中断例程，其中`0x15`号中断提供了内存探测功能，物理内存容量的探测正是籍由它实现的。

2. 在Intel推出80286时，为了兼容过去的20位地址线（A0~A19），默认将A20以及其上的地址线封锁了，要启用32位地址线，得先设置一个端口中的标志位，这一操作称为“打开A20地址线”。

3. 全局描述符表是由段描述符构成的一个数组，每个段描述符都描述了一种访问内存的“View”，譬如访问范围的开始和结尾、内存是否只读等。TinyOS的bootloader只准备了两个段，分别是覆盖了整个地址空间的代码段和数据段。前者具有可执行属性，用于填充cs段寄存器；后者是可读写的，用于填充ds，gs，ss等一票寄存器。

4. Intel推出80286后，为了兼容之前的16位计算机，将默认进入的16位运行模式称为“实模式”。与之相对的，是更为先进的“保护模式”。要进入保护模式，需要将cr0寄存器中的一个标志位置为1。

5. 要启用分页，需要先填充初始的页目录和页表（x86采用两级页表，每个页为4KB，一个页表包含1024个页的物理地址，页目录则包含1024个页表的物理地址），而填充的内容决定了进入虚拟地址空间后的内存布局。对此，bootloader将物理内存的0~4MB占用，并给出了以下从虚拟地址到物理地址的转换：

   ```
   0x00000000~0x003fffff => 0x00000000~0x003fffff
   0xc0000000~0xc03fffff => 0x00000000~0x003fffff
   ```
   
   这4MB空间用途如下：
   
   `0x00000000~0x000fffff`和实模式下的用处相同，为内核代码和显存区域
   
   `0x00100000~0x001fffff`为内核栈区域，即内核进程所使用的栈（挺奢侈的）
   
   `0x00200000~0x002fffff`为内核虚拟地址空间的页目录以及前255个页表
   
   `0x00300000~0x003fffff`为内核保留区域

6. 之前提到，系统内核文件kernel.bootbin是ELF32格式的，因此bootloader需要将其执行视图的各个段装载到虚拟地址空间中正确的位置，并跳转到内核的入口地址处执行。

从进入内核开始，绝大部分的功能就能用C语言编写了。我原本就对x86汇编不熟悉，编写MBR和bootloader时寸步难行，东拼西凑，几乎就要在内核门口放弃了。可是如今看来却又不足道也，大概受苦都是这样吧。

## 内核内存管理

### 多级位图记录物理页使用情况

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

![]({{site.url}}/postpics/multi-level-bitmap.png){: width="70%"}

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
/* In kernel/memory/phy_mem_man.h */
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

当然，位图本身也要占用一定的空间，TinyOS将它放置在物理内存中`0x00300000~0x003fffff`这段内核保留区域的开头处。

### 分配和释放物理页

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
/* In kernel/memory/phy_mem_man.h */
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
/* In kernel/memory/phy_mem_man.h */
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

### 内核页目录和页表

自从bootloader开启了分页机制以来，内核就一直运行在自己的虚拟地址空间中。这里复述一下，初始的页表和页目录是这样安排的：

![]({{site.url}}/postpics/init-PDE-PTE.png){: width="90%"}

可以看到，在初始化分页机制时，页目录的高255项就被指向了255个页表，尽管实际映射了物理页的只有首个页表，这么做算是个伏笔。众所周知，32位linux将0~3GB地址空间分配给进程使用，每个进程都独占自己0~3GB的虚拟地址空间；而3GB~4GB空间则被所有进程共享，是内核占用的地址空间。这一共享在x86上的实现方式就是共享页表——3GB~4GB空间对应于页目录的第768~1023项，于是我干脆预先分配了255个页表，以后所有进程页目录的第768~1022项都指向这255个页表，这样就实现了高地址空间的共享。

另一个值得注意的细节是页目录的第1023项指向了它自身而非一个页表。通常，页目录项应该存放页表的物理地址，这里把页目录当作页表来访问是一个小trick，它使得我们不必通过页目录和页表的虚拟地址就能在开启分页的情况下直接访问页目录、页表的内容。设想我希望访问页目录中第`i`项内容，那么可以按如下方式构造一个虚拟地址——

{% highlight c %}
uint32_t make_iPDE_addr(uint32_t i)
{
    ASSERT_S(i < 1024);
    return (0b1111111111 << 22) | (0b1111111111 << 12) | (i << 2);
}
{% endhighlight %}

这里`0b1111111111`使用了GCC的二进制字面量扩展。在x86的二级页表方案下，一个虚拟地址被划分为三部分，其中22~31位为页目录索引项，12~21位为页表索引项，0~11位为页内偏移。据此，将虚拟地址转换为物理地址的伪代码为（假设页表和页表项都存在，不会发生缺页中断）：

{% highlight c %}
uint32_t vir_2_phy(uint32_t v)
{
    // PDE: page directory entry，即页目录
    // PTE: page table entry，即页表
    PTE *pte = pde->entrys[v >> 22];
    return pte->entrys[(v >> 12) & 0b1111111111] + (v & 0xfff);
}
{% endhighlight %}

现在把`make_iPDE_addr(i)`构造出的地址带入`vir_2_phy`中，我们将会发现得到的正是页目录第`i`项的物理地址。类似地，可以用下面的方式得到页目录第`i`项对应的页表中的第`j`项的虚拟地址：

{% highlight c %}
uint32_t make_iPTE_addr(uint32_t i, uint32_t j)
{
    ASSERT_S(i < 1024 && j < 1024);
    return (0b1111111111 << 22) | i | (j << 2);
}
{% endhighlight %}

以后每个进程都有自己的页目录，它们的页目录第1023项也都会指向自身所在页的物理地址。

### 虚拟地址空间

TinyOS的每个进程有自己的虚拟地址空间，而隐藏在这一美好叙述下的却是丑陋且繁琐的页目录和页表操作。`include/kernel/memory/vir_mem_man.h`提供了以下接口，让进程管理模块不需要考虑与分页有关的琐事：

{% highlight c %}
/*
    创建一个虚拟地址空间并返回其句柄
    初始虚拟地址空间中有些部分是已经被用了的，使用情况如下：
        3GB~(4GB - 4M)永远和内核共享
        最高的页目录项指向页目录自身所在物理页（虽然按理说它应该指向一个页表）
    创建的页目录本身的虚拟地址在内核共享区域内，且使用的物理页是常驻内存的
*/
vir_addr_space *create_vir_addr_space(void);
/* 将某个虚拟地址空间设置为当前正使用 */
void set_current_vir_addr_space(vir_addr_space *addr_space);

/* 取得当前虚拟地址空间句柄 */
vir_addr_space *get_current_vir_addr_space(void);

/* 取得自bootloader开始内核所使用的虚拟地址空间句柄 */
vir_addr_space *get_ker_vir_addr_space(void);

/* 销毁某个虚拟地址空间，其占用的所有物理页均将被释放 */
void destroy_vir_addr_space(vir_addr_space *addr_space);
{% endhighlight %}

所有的页目录虚拟地址都在3GB以上的地址空间中，因此在任意虚拟地址空间内，TinyOS都能够访问任意一个页目录的内容。

x86 CPU有一个名为`cr3`的寄存器，它存放的是当前使用的页目录的物理地址。只要修改该寄存器的值，就能切换当前使用的`虚拟地址->物理地址`映射。事实上，对每个进程，TinyOS都会为它初始化一个页目录，并在该进程的线程被调度执行时把`cr3`赋为其页目录的地址。

由于访问页目录项必须通过虚拟地址来完成，而设置`cr3`寄存器又要用到它的物理地址，因此TinyOS用一个二元组`(页目录虚拟地址，页目录物理地址)`来作为一个虚拟地址空间的`handle`：

{% highlight c %}
/* In kernel/memory/vir_mem_man.h */
struct PDE_struct
{
    uint32_t PTE[1024];
};

struct PTE_struct
{
    uint32_t page[1024];
};

struct _vir_addr_space
{
    struct PDE_struct *vir_PDE;
    uint32_t phy_PDE;
};

typedef struct _vir_addr_space vir_addr_space;
{% endhighlight %}

要创建一个虚拟地址空间并返回其handle，其实就是分配一个页面，把它当作页目录来填充内容，最后返回其虚拟地址和物理地址即可：

{% highlight c linenos %}
/*
    这里的empty_usr_addr_space_rec是一个用来存放尚可用的 struct _vir_addr_space 的自由链表
    TinyOS预先分配好了固定数量的 struct _vir_addr_space 空间，需要的时候从其中取走即可，写起来方便（逃
    当然，这样的设计也浪费了一些空闲空间，且限制了页目录的最大数量

    In kernel/memory/vir_mem_man.h
*/
static vir_addr_space *new_empty_usr_addr_space(void)
{
    if(!empty_usr_addr_space_rec)
        return NULL;

    if(empty_usr_addr_space_rec->next)
        empty_usr_addr_space_rec->next->last = NULL;
    
    vir_addr_space *rt = (vir_addr_space*)empty_usr_addr_space_rec;
    empty_usr_addr_space_rec = empty_usr_addr_space_rec->next;

    return rt;
}

vir_addr_space *create_vir_addr_space(void)
{
    // 分配一个虚拟地址空间记录
    vir_addr_space *rec = new_empty_usr_addr_space();
    if(!rec)
        return NULL;
    
    // 取得一个页目录的虚拟地址
    rec->vir_PDE = get_page_of_usr_addr_space(rec);
    // 取得页目录对应的物理地址
    rec->phy_PDE = vir_to_phy((uint32_t)rec->vir_PDE);

    // 清空没用到的页目录
    for(size_t i = 0;i != 768; ++i)
        rec->vir_PDE->PTE[i] = 0;

    // 768-1022项与内核共享
    for(size_t i = 768; i != 1022; ++i)
        rec->vir_PDE->PTE[i] = ker_addr_space.vir_PDE->PTE[i];
    
    // 最后一项指向自身所在页框的物理地址
    rec->vir_PDE->PTE[1023] = iPDE(rec->phy_PDE,
                                   PAGE_PRESENT_TRUE,
                                   PAGE_RW_READ_WRITE,
                                   PAGE_USER_USER);
    
    return rec;
}
{% endhighlight %}

虚拟地址空间的销毁也是简单粗暴，把页目录中所有用到的页表以及这些页表中记录的物理页地址挨个让物理内存管理系统释放即可，这里就不放代码了。

至于切换当前使用的虚拟地址空间，修改`cr3`寄存器即可——

{% highlight c %}
/* In include/kernel/asm.h */
static inline void _load_cr3(uint32_t val)
{
    asm volatile ("movl %0, %%cr3" : : "r" (val) : "memory");
}

/* In kernel/memory/vir_mem_man.c */
void set_current_vir_addr_space(vir_addr_space *addr_space)
{
    cur_addr_space = addr_space; // cur_addr_space是用来记录当前使用的虚拟地址空间的全局变量
    _load_cr3(addr_space->phy_PDE);
}
{% endhighlight %}

### 缺页中断处理

在开启了分页机制的情况下，访问内存时使用的地址都要经过专门的硬件翻译为虚拟地址，这一翻译过程是通过查询页目录和页表实现的。到目前为止，我们访问的所有地址对应的页目录和页表都被预先填充好了。然而页目录中还有许多空白项，如果访问这些地址，CPU就会产生一个缺页异常（page fault）打断当前执行的程序，让对应的中断处理函数来填充对应的页目录/页表项，然后从导致缺页异常的指令处恢复执行。

中断处理系统的搭建我尚未介绍，但现在这不是重点。假设我们已经准备好了一套中断处理框架，那么缺页中断（虽然是fault，但我总觉得叫缺页中断比较顺口……）函数应该由以下几个步骤构成：

1. 获取是哪个虚拟地址导致了缺页中断
2. 检查该虚拟地址在页目录中的页表是否缺失，若是，为其分配页表
3. 给缺失的页面分配一个物理页框，填充到对应页表中
4. 刷新TLB条目

x86中导致缺页中断的虚拟地址会被存放到`cr2`寄存器中。万事俱备，缺页终端处理的C语言实现如下：

{% highlight c %}
/* In kernel/memory/vir_mem_man.h */
void page_fault_handler(void)
{
    // 取得cr2寄存器内容，即引发pagefault的虚拟地址
    uint32_t cr2 = _get_cr2();

    // 计算页表索引和物理页索引
    uint32_t PTE_idx = cr2 >> 22;
    uint32_t page_idx = (cr2 >> 12) & 0x3ff;

    // 若页表不存在，就分配一个
    struct PTE_struct *PTEaddr = make_PTE_vir_addr(PTE_idx);
    if(!(cur_addr_space->vir_PDE->PTE[PTE_idx] & PAGE_PRESENT_TRUE))
    {
        cur_addr_space->vir_PDE->PTE[PTE_idx] = iPDE(
                        alloc_phy_page(true),
                        PAGE_PRESENT_TRUE,
                        PAGE_RW_READ_WRITE,
                        PAGE_USER_USER);
        
        // 清空新的页表
        memset((char*)PTEaddr, 0, 4096);
    }

    // 给缺失页面分配一个物理页
    // 缺页中断分配的结果都是非常驻内存的，常驻内存的页必须是内核预先分配的页
    PTEaddr->page[page_idx] = iPTE(
                    alloc_phy_page(false),
                    PAGE_PRESENT_TRUE,
                    PAGE_RW_READ_WRITE,
                    PAGE_USER_USER);
    
    // 刷新TLB
    _refresh_vir_addr_in_TLB(cr2);
}
{% endhighlight %}

## 中断和系统调用

### 中断描述符表

x86规定使用中断描述符表（Interrupt Descriptor Table，IDT）来描述中断处理函数的入口，中断描述符表是以中断描述符为元素类型的数组，我们只需要在内存中填充好IDT，然后使用`lidt`指令告知机器IDT的位置和大小即可。

每个中断描述符占64字节，包含了以下信息：

1. 中断处理函数所处的段
2. 中断处理函数的段内偏移
3. 中断处理函数的访问属性，如DPL特权级等

假设已经准备好了一个IDT数组，那么`IDT[0]`就给出了0号中断处理函数的入口，`IDT[1]`给出了1号中断处理函数，以此类推。x86规定了一部分中断号的含义，其他中断号则留给操作系统分配——

```
中断号   中断类型
    0   除0异常
    1   DEBUG
    2   NMI致命错误
    3   断点
    ......
   13   GP保护错误
   14   缺页
    ......
   31   保留
```

可以看到，0～31号中断已经有了自己的含义，TinyOS按照其类型安排合适的中断处理函数，譬如`IDT[14]`就应该描述缺页中断处理函数的入口。

### 8259A

8259A芯片并不是x86 CPU的一部分，几乎所有的外部设备中断都是经由它处理后发送给CPU的。在这里我不打算讨论其引脚映射和多个8259A的级联，只需要知道：可以通过向特定的端口发送数据来设置8259A向CPU发送的中断号范围。譬如，下面的代码将8259A发送的中断号设置为从32开始：

{% highlight c %}
// 主片
_out_byte_to_port(0x20, 0x11);
_out_byte_to_port(0x21, 0x20);
_out_byte_to_port(0x21, 0x04);
_out_byte_to_port(0x21, 0x01);
// 从片
_out_byte_to_port(0xa0, 0x11);
_out_byte_to_port(0xa1, 0x28);
_out_byte_to_port(0xa1, 0x02);
_out_byte_to_port(0xa1, 0x01);
{% endhighlight %}

此时8259A覆盖的中断号范围为32~47：

```
中断号   中断类型
   32   时钟
   33   键盘
    ......
   46   硬盘
   47   保留
```

### 系统调用

TinyOS的系统调用机制模仿了Linux——使用0x80号中断实现。通常，系统调用由用户进程发起，且往往涉及到比用户进程特权级更高的操作，而发起中断就是最简单的提升特权级的方法之一。系统调用通常涉及到一个或多个参数传递，在TinyOS中采用约定的寄存器进行传参。总而言之，用户程序只需要将特定寄存器设置为要传入的参数，然后执行指令`int 0x80`即可。

作为系统调用入口的中断处理函数与普通的中断处理函数入口并无太大不同，唯一的区别就是把约定好的寄存器值压入栈中作为系统调用函数实现的参数。中断处理函数入口和系统调用入口均实现在`kernel/interrupt/interrupt.s`中，就不在这里赘述过多的技术细节了。



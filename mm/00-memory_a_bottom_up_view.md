内存模块是内核中一个非常重要的部分。我们现在的计算机都被称作为**存储程序计算机**，也就是所有待运行的程序和数据都需要加载在内存当中方能被执行。正因为如此，很多朋友都希望学习内存模块的工作机制，但碍于内核代码的庞大以及文档的缺失和稀少，总是感觉无从下手。

经过一段时间的摸索，我终于对内存模块有了一点点的了解。今天整理成文，希望能给想要探究内存模块的朋友一点点借鉴。

# 内存模块的层次结构

首先内存模块具有一定层次结构的，从物理内存到软件控制的内存经过了几个层次的隔离。

[e820从硬件获取内存分布][1]

[原始内存分配器--memblock][2]

[页分配器][22]

[Slub分配器][23]

大致我们能看到这么四个层次的内存管理结构。前两者基本在内核启动时使用，而平时大多使用的是后两者。

# 内存管理的不同粒度

越是学习，越发现内存管理是一个比较大的话题。这么一路走来，我们从最底层的物理信息一直走到了slub分配器，可以说对内存管理有了一定的认知。

此刻请允许我站的远一点来观察整个内存管理架构，也从另一个角度 -- **粒度** 来观察内存管理的奥妙。

[内存管理的不同粒度][21]

# 挑战和进化

内存管理的层次结构已经逐渐清晰，接着发现在内存子系统随着应用场景和硬件环境的变化也会遇到新的挑战，并对此做出自身的进化。

在此仅以一点点的记录来进一步窥探这个神秘的世界。

[挑战和进化][11]

# 参考文献

[Understand Linux VM][18]

[1]: /mm/01-e820_retrieve_memory_from_HW.md
[2]: /mm/02-memblock.md
[3]: /mm/03-sparsemem.md
[5]: /mm/05-Node_Zone_Page.md
[6]: /mm/06-page_alloc.md
[7]: /mm/07-per_cpu_pageset.md
[8]: /mm/08-slub_general.md
[9]: /mm/09-slub_in_graph.md
[10]: /mm/10-page_struct.md
[11]: /mm/50-challenge_evolution.md
[18]: https://www.kernel.org/doc/gorman/html/understand/index.html
[19]: /mm/11-users_of_buddy.md
[20]: /mm/12-gfp_usage.md
[21]: /mm/13-physical-layer-partition.md
[22]: /mm/page_allocator/00_page_allocator.md
[23]: /mm/slub_allocator/00_slub.md
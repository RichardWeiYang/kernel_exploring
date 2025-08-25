一般大家都比较了解内存分配的伙伴系统，但是你知道吗，当内核刚启动的时候，伙伴系统还没有准备好，是不能使用的。这个时候就有另一个简单的内存分配器--memblock。

# 看一眼历史

memblock也不是内核的原配，在memblock之前还有其他的初期内存分配器，比如bootmem。memblock是在2010年Yinghai提出的。有兴趣的可以看一下当时的[邮件列表][1]中的讨论。

而在代码中，应该是这个commit引入了memblock。

[95f72d1ed41a66f1c1c29c24d479de81a0bea36f][2]

可以看到，memblock以前叫lmb，而这个补丁只是改了下名字。

而这个lmb是Paul Mackerras在2005年写的

[7c8c6b9776fb41134d87ef50706a777a45d61cd4][4]

真是好古早哦。

不过现在的实现和当年的实现可以说是千差万别了。当前的实现是Tejun Heo重新实现的。

[784656f9c680d334e7b4cdb6951c5c913e5a26bf][5]

# 整体架构


memblock管理了两段区域：memblock.memory和memblock.reserved。

所有物理上可用的内存区域都会被添加到memblock.memory。而被分配或者被系统占用的区域则会添加到memblock.reserved。

**注意: 被分配的内存空间并不会从memblock.memory区域中移除。**

让我借用一张图来解释一下，这张图的原版在[这个系列][3]，也是我非常喜欢的一个内核探索的系列。

```
+---------------------------+   +---------------------------+
|         memblock          |   |       Array of the        |
|  _______________________  |   |      memblock_region      |
| |        memory         | |   |                           |
| |      memblock_type    |-|-->| [start1, end1)            |
| |_______________________| |   |                           |
|                           |   | [start2, end2)            |
|                           |   |                           |
|                           |   | [start3, end3)            |
|                           |   |                           |
|                           |   +---------------------------+
|                           |                                
|  _______________________  |   +---------------------------+
| |       reserved        | |   |       Array of the        |
| |      memblock_type    |-|-->|      memblock_region      |
| |_______________________| |   |                           |
|                           |   |  [start1, end1)           |
|                           |   |                           |
|                           |   |  [start2, end2)           |
|                           |   |                           |
+---------------------------+   +---------------------------+
```

在上图的这个例子中，当前memblock的状态是：
1. 当前系统中各有三段可用的内存空间 [start1/2/3, end1/2/3)
2. 而其中的两个已被分配 [start1/2, end1/2)

假如你想要释放 [start2, end2), 那么memblock.memory并不会有什么变化, 只要从memblock.reserved中移除 [start2, end2) 就好。

希望通过这张简单的图能帮助你理解memblock的运作原理。

PS:其实我隐藏了一个很重要的信息，不过现在暂时不需要看到他。待到春花烂漫，你们自然就会相见。

# 初始化流程

之前我们已经看过，内存的信息通过e820从硬件中获取保存在了相应的结构体中。那现在的问题就是memblock是怎么对应上实际的物理内存的呢？

全局流程如下：

```
start_kernel()
    setup_arch()
        e820__memory_setup()
            max_pfn = e820__end_of_ram_pfn()
        memblock_set_current_limit(ISA_END_ADDRESS)
        e820__memblock_setup()
            memblock_add()
                memblock_add_range(&memblock.memory, base, size, MAX_NUMNODES, 0)
            memblock_dump_all()
        init_mem_mapping()                           // 设置内核页表(direct mapping)
            end = max_pfn << PAGE_SHIFT;
            memory_map_bottom_up(kernel_end, end);
                max_pfn_mapped = max(max_pfn_mapped, end_pfn)
        memblock_set_current_limit(get_max_mapped())

        initmem_init() -> x86_numa_init() -> numa_init()
            memblock_set_node(0, ULLONG_MAX, &memblock.memory, MAX_NUMNODES) // 默认都先归到node MAX_NUMNODES
            memblock_set_node(0, ULLONG_MAX, &memblock.reserved, MAX_NUMNODES)
            numa_register_memblks()
                memblock_set_node()                  // 再根据numa信息设置真实的node
                alloc_node_data(nid)                 // allocate pgdata for each node with memory
                    node_set_online(nid)             // 因为只有有内存的node才会分配，所以这里online的都是有内存的

        dma_contiguous_reserve(max_pfn_mapped..)     // 给dma分好cma区域,好像把所有的内存都包近来了

        x86_init.paging.pagetable_init() -> paging_init()
            sparse_init()
                memblock_presents()
                    memory_present()                 // 分配mem_section，并标记present
                sparse_init_nid()
                    sparse_usage_init()              // 分配usage空间，包含subsection和pageblock
                    __populate_section_memmap()      // 拿到mmep，也就是当前section的struct page的内存
                    sparse_init_early_section()      // struct page设置到mem_section->section_mem_map，usage设置到usage(包括pageblock)
            zone_size_init()
                free_area_init()                     // 初始化pgdat
                    free_area_init_node()            // call for each pgdat
                    node_set_state(nid, N_MEMORY)    // 存在可用内存的节点标上MEMORY
                    memmap_init()->memmap_init_range // 初始化page struct，__init_single_page()
                        defer_init()                 // 对应的page延后初始化
                        init_pageblock_migratetype() // 设置默认migratetype为MIGRATE_MOVABLE
    mm_core_init()
        build_all_zonelists(NULL)                    // 构造page allocator的zonelist
        memblock_free_all()                          // 将内存释放到buddy
            reset_all_zones_managed_pages()
            free_low_memory_core_early()
                memmap_init_reserved_pages()         // 设置PageReserved
                    memblock_set_node()              // 全量设置memblock.reserved中的nid信息
                    reserve_bootmem_region()         // 初始化预留页的page（不受defer_init影响）
                __free_memory_core(start, end)       // release free pages to buddy
                    memblock_free_pages( , order)    //
                        early_page_initialised(pfn)  // 如果设置了defer init，则会推迟释放
                        __free_pages_core()          // 真释放
        mem_init()
        kmem_cache_init()                            // 可以用slab了
    rest_init()
        kernel_init()
            kernel_init_freeable()
                smp_init()                           // 在这之前只有单线程, memblock分配只能发生在这之前
                page_alloc_init_late()
                    deferred_init_memmap()           // 延迟初始化 page struct，并释放到buddy
                    memblock_discard()               // discard region array
                do_basic_setup()
                    do_initcalls()
                        cma_init_reserved_areas()    // 释放cma到buddy
                    init_per_zone_wmark_min()
                        setup_per_zone_wmarks()      // 初始化wmark
            free_initmem()
                free_kernel_image_pages()            // 释放__init标记的，在__init_begin/end之间的内存
```

在x86平台，这个工作就交给了 e820__memblock_setup()，从e820信息中构建了memblock。

```
void __init e820__memblock_setup(void)
{
    int i;
    u64 end;

    /*
     * EFI may have more than 128 entries
     * We are safe to enable resizing, beause memblock_x86_fill()
     * is rather later for x86
     */
    memblock_allow_resize();

    for (i = 0; i < e820.nr_map; i++) {
        struct e820entry *ei = &e820.map[i];

        end = ei->addr + ei->size;
        if (end != (resource_size_t)end)
            continue;

        if (ei->type != E820_RAM && ei->type != E820_RESERVED_KERN)
            continue;

        memblock_add(ei->addr, ei->size);
    }

    /* throw away partial pages */
    memblock_trim_memory(PAGE_SIZE);

    memblock_dump_all();
}
```

通过他就建立了硬件信息和memblock之间的联系。在x86平台上，不断从e820中获取内存底层信息，并添加到memblock中。你看是不是很简单了。

## 增加numa信息

[NUMA(Non-uniform memory access)][6]是一个高级玩意，咱先来看看维基百科上的定义：

is a computer memory design used in multiprocessing, where the memory access time depends on the memory location relative to the processor. Under NUMA, a processor can access its own local memory faster than non-local memory (memory local to another processor or memory shared between processors). The benefits of NUMA are limited to particular workloads, notably on servers where the data are often associated strongly with certain tasks or users.

总的一句话来讲，就是分配内存时需要区别对待。不同区域的内存我们叫做不同的节点，节点之间我们又定义了个“距离”的概念用来衡量内存访问时间的差异。

好了，有了基本的概念，我们来看看linux内核中是如何获得物理内存的NUMA信息。说白了就是怎么知道一块内存对应到哪个NUMA节点。

在x86平台，这个工作分成两步

* 将numa信息保存到numa_meminfo
* 将numa_meminfo映射到memblock结构

所以，前面我们看到的memblock信息上是没有numa信息的，在x86上这部分信息是等到numa初始化时再把这部分信息加到memblock中的。

在上面的流程中，我们可以搜x86_numa_init()，看看这部分初始化在全局中的顺序。

### 将numa信息保存到numa_meminfo

在x86架构上，numa信息第一次获取是通过acpi或者是读取北桥上的信息。具体的函数是numa_init()。不管是哪种方式，numa相关的信息都最后保存在了numa_meminfo这个数据结构中。

这个数据结构和memblock长得很像，展开看就是一个数组，每个元素记录了一段内存的起止地址和node信息。

```
    numa_meminfo
    +------------------------------+
    |nr_blks                       |
    |    (int)                     |
    +------------------------------+
    |blk[NR_NODE_MEMBLKS]          |
    |    (struct numa_memblk)      |
    |    +-------------------------+
    |    |start                    |
    |    |end                      |
    |    |   (u64)                 |
    |    |nid                      |
    |    |   (int)                 |
    +----+-------------------------+
```

在这个过程中使用的就是这个函数添加的numa_meminfo数据结构。

* numa_add_memblk()

这个函数非常简单，而且粗暴。

### 将numa_meminfo映射到memblock结构

内核获取了numa_meminfo之后并没有如我们想象一般直接拿来用了。虽然此时给每个numa节点生成了我们以后会看到的node_data数据结构，但此时并没有直接使能它。

memblock是内核初期内存分配器，所以当内核获取了numa信息首先是把相关的信息映射到了memblock结构中，使其具有numa的knowledge。这样在内核初期分配内存时，也可以分配到更近的内存了。

在这个过程中有两个比较重要的函数

* numa_cleanup_meminfo()
* numa_register_memblks()

前者主要用来过滤numa_meminfo结构，合并同一个node上的内存。

后者就是把numa信息映射到memblock了。除此之外，顺便还把之后需要的node_data给分配了，为后续的页分配器做好了准备。

### 观察memblock的变化

memblock的调试信息默认没有打开，所以要观察的话需要传入内核启动参数"memblock=debug"。

进入系统后，输入命令"dmesg | grep -A 9 MEMBLOCK"可以看到

```
[    0.000000] MEMBLOCK configuration:
[    0.000000]  memory size = 0x00000001f9c4e800 reserved size = 0x0000000014aab27c
[    0.000000]  memory.cnt  = 0x7
[    0.000000]  memory[0x0]	[0x0000000000001000-0x000000000009cfff], 0x000000000009c000 bytes flags: 0x0
[    0.000000]  memory[0x1]	[0x0000000000100000-0x00000000ba5b1fff], 0x00000000ba4b2000 bytes flags: 0x0
[    0.000000]  memory[0x2]	[0x00000000ba5b9000-0x00000000bad8dfff], 0x00000000007d5000 bytes flags: 0x0
[    0.000000]  memory[0x3]	[0x00000000bafb6000-0x00000000ca8a1fff], 0x000000000f8ec000 bytes flags: 0x0
[    0.000000]  memory[0x4]	[0x00000000ca93a000-0x00000000ca977fff], 0x000000000003e000 bytes flags: 0x0
[    0.000000]  memory[0x5]	[0x00000000cafff000-0x00000000caffffff], 0x0000000000001000 bytes flags: 0x0
[    0.000000]  memory[0x6]	[0x0000000100000000-0x000000022f5fffff], 0x000000012f600000 bytes flags: 0x0
--
[    0.000000] MEMBLOCK configuration:
[    0.000000]  memory size = 0x00000001f9c4e800 reserved size = 0x0000000014b29800
[    0.000000]  memory.cnt  = 0x7
[    0.000000]  memory[0x0]	[0x0000000000001000-0x000000000009cfff], 0x000000000009c000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x1]	[0x0000000000100000-0x00000000ba5b1fff], 0x00000000ba4b2000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x2]	[0x00000000ba5b9000-0x00000000bad8dfff], 0x00000000007d5000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x3]	[0x00000000bafb6000-0x00000000ca8a1fff], 0x000000000f8ec000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x4]	[0x00000000ca93a000-0x00000000ca977fff], 0x000000000003e000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x5]	[0x00000000cafff000-0x00000000caffffff], 0x0000000000001000 bytes on node 0 flags: 0x0
[    0.000000]  memory[0x6]	[0x0000000100000000-0x000000022f5fffff], 0x000000012f600000 bytes on node 0 flags: 0x0
```

注意看每个memory条目的最后，现在每个memblock上都多了一个输出"on node 0"。这就是物理内存的NUMA信息。当然我这里只有一个node，如果你的系统上有多个node那就可以看到不一样的信息了。

#### 插播一条我的补丁

在code review过程中，发现numa_nodemask_from_meminfo()的作用其实可以被别的东西替代。所以我干脆提了个补丁把这个挪走了。

补丁的id是：[474aeffd88b87746a75583f356183d5c6caa4213][7]

不过可惜的是，这个补丁还有点小问题。新的改进也写好了，但是到现在还没有合入。社区嘛，你懂的。

### 重要的全局变量

#### node_stats[NR_NODE_STATES]

这是一个nodemask_t类型的数组，其中每一个数组表达一个纬度的信息。

```
enum node_states {
	N_POSSIBLE,		/* The node could become online at some point */
	N_ONLINE,		/* The node is online */
	N_NORMAL_MEMORY,	/* The node has regular memory */
#ifdef CONFIG_HIGHMEM
	N_HIGH_MEMORY,		/* The node has regular or high memory */
#else
	N_HIGH_MEMORY = N_NORMAL_MEMORY,
#endif
	N_MEMORY,		/* The node has memory(regular, high, movable) */
	N_CPU,		/* The node has one or more cpus */
	N_GENERIC_INITIATOR,	/* The node has one or more Generic Initiators */
	NR_NODE_STATES
};
```

其中有两个常用的，被单独重命名。

```
#define node_online_map 	node_states[N_ONLINE]
#define node_possible_map 	node_states[N_POSSIBLE]
```

#### node_data[MAX_NUMNODES]

每个numa节点的pgdata，就是后面内存管理的重要数据结构。

## 剔除已经占用的部分

随着代码的阅读，发现之前遗漏了非常重要的一部分。

当我们从e820上报的内存信息构建memblock信息时，我们的内核其实已经占用了一部分物理内存。所以如果我们不加处理，那内核已经使用的物理内存就会被当作可用部分而挪作他用。当然后果可想而知。

接下来，我们尝试看看内核初始化过程中，有哪些地方剔除已经占用的物理内存。

```
start_kernel()
    setup_arch()
        early_reserve_memory()
            memblock_reserve(__pa_symbol(_text), )
            memblock_reserve(0, SZ_64K)
            early_reserve_initrd()
            memblock_x86_reserve_range_setup_data()
            reserve_bios_regions()

        e820__memory_setup()

        e820_add_kernel_range() // 把[_text, _end]从e820里去除了

        reserve_brk()
            memblock_reserve(__pa_symbol(_brk_start), )

        memblock_set_current_limit(ISA_END_ADDRESS)
        e820__memblock_setup()
```

从上面的流程可以看出，在真正建立memblock信息之前，已经把占用的内存给reserve了。

不过里面有意思的是，有些地方是先reserve memblock，再清理了e820的信息，比如e820_add_kernel_range。理论上e820上没有标明可用的内存是不会在memblock里出现的。可能是为了在memblock中留个念想吧。

# 对外的API

对memblock有了大致的了解，你或许会想知道相关的api是什么样子的，是怎么使用的。

## memblock_add_[node]/memblock_remove

使用memblock的第一步就是要从下一层中获取可用的内存区域并填写到memblock.memory中。这是通过memblock_add() and memblock_remove()实现的。而且这两个函数保证运行后memblock中的区域是排序的。

**注意：这两个函数只改变memblock.memory。**

## memblock_alloc/memblock_free

memblock的重要作用就是内核初期的内存分配器了。那通过memblock来分配释放内存分别通过memblock_alloc() and memblock_free()来实现。

**注意： 这两个函数只改变memblock.reserved。**

memblock_phys_alloc 返回的是物理地址
memblock_alloc      返回的是虚拟地址

# 具体实现

## memblock_add_range()

这个函数用来添加一个区域到membloc中，比如添加一个硬件的内存区域到memblock.memroy。

实现的方法比较简单，添加再合并。总的来说一共有四种情况。

* total covered
* no overlap
* partial overlapped – beginning
* partial overlapped – end

让我们来挨个儿解释。

图例：
(base, end) 表示要添加的新的内存区域.
[rbase, rend] 表示已有的内存区域.

### total coverd

```
           (base,     end)
      [rbase,             rend]
```

在这种情况下，除了检查区域的属性，其他不会做。

### no overlap

```
                                              (base,     end)
                      [rbase,             rend]

或者

    (base,     end)
                      [rbase,             rend]
```

这种情况也简单， 直接添加就好。另外，这第二个情况说明已经遍历完现有的区域，可以停止了。

### partial overlapped – beginning

```
   (base,                  end)
               [rbase,             rend]

   After insert the lower part.

               ||
               ||
               vv

   （baes, end'）          (end)
               [rbase,             rend]
```

在这种情况下， (base, rbase-1) 将会被添加到区域中。此后base被设置为end。这样就回到了第一个total cover的情况。

### partial overlapped – end

```
                        (base,          end)
        [rbase,             rend]

   After truncate the lower part.

               ||
               ||
               vv

                                (base', end)
        [rbase,             rend]
```

这种情况下(base, rend)部分将会被忽略，而base会直接赋值为rend。这样也就回到了第二种情况 no overlap。

## memblock_remove_range()

这是add range的另一半。

## memblock_isolate_range()

这个函数在多个地方使用到，比如memblock_remove_range() 和memblock_set_node()。这个函数的目的就是按照要求将memblock中的区域划分开。

```
                         (base,                         end)

      [rbase1,                   rend1]    [rbase2,                  rend2]

           After split

               ||
               ||
               vv

                         (base,                         end)

      [rbase1,    rend1'][rbase1',rend1]   [rbase2,   rend2'][rbas2', rend2]


                          start_rgn         end_rgn
```

该函数一共有五个参数

* type  指定是memblock.memory或memblock.reserved
* base/size 指定像划分的那段区域的范围
* start_rgn/end_rgn  返回划分完后这段区域对应的编号

其实就是划分，划分完了返回被划分的那段区域的编号～

## memblock_find_in_range_node

通过for_each_free_mem_range_reverse在指定的区域，找到空闲的内存空间。

## memblock_alloc_range_nid

这是各种memblock_alloc的最终内部实现。

主要调用了memblock_find_in_range_node找到空闲的空间。

# 各种遍历

## for_each_mem_pfn_range

单纯遍历memblock.memory，只要对应的区域大小超过一个page，就返回该区域的首末地址的pfn和nid。

值得注意的是返回的end_pfn实际上是不属于这个区域的。

## for_each_free_mem_range

同时遍历memblock.memory和memblock.reserved，并找出在memory中，但是不在reserved中的区域。

这样可以找出空闲的内存区域。

同样，这里返回的end也不属于当前区域范围。

# 调试和测试

## 调试

kernel parameter中加上 memblock=debug，可以开启debug模式。

* 在dmesg中打印相关调试信息，在dmesg中搜"MEMBLOCK configuration"
* 在debugfs中可以查看各个类型信息

不过这个debugfs要配置上了后，才能看到。

## 测试

memblock作为启动初期的代码，不是很好调试。所以社区在tools/testing/memblock目录下，有针对memblock的测试代码。如果有什么memblock的改动，建议先运行测试程序，确保测试程序无误。

进入该目录后直接make，生成测试程序，然后直接运行./main。

也可以 make NUMA=1，测试有numa的情况。


[1]: https://lkml.org/lkml/2010/7/13/114
[2]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/mm/memblock.c?id=95f72d1ed41a66f1c1c29c24d479de81a0bea36f
[3]: https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html
[4]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7c8c6b9776fb41134d87ef50706a777a45d61cd4
[5]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=784656f9c680d334e7b4cdb6951c5c913e5a26bf
[6]: https://en.wikipedia.org/wiki/Non-uniform_memory_access
[7]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=474aeffd88b87746a75583f356183d5c6caa4213

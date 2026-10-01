Node, Zone, Page是内核中内存管理模块非常常见的几个概念，但是一直以来都不清楚其究竟是什么含义，之间又有什么联系。我想很多人也有类似我的困惑。今天我就装个大头，帮助大家理理这里面的关系。

# 一张经典的图

![这里写图片描述](https://www.kernel.org/doc/gorman/html/understand/understand-html001.png)

此图来自[Describing Physical Memory][1]

这张图想必有探索过内存管理的同学可能都会见过，但是估计大部分看不懂，或者知道个一知半解。因为我以前看的时候也看不懂，这讲的是个毛？好像是有点那个意思，但是是如何和实际上的内存对应起来的呢？

希望看完这篇文章后，能对这张图能够有更多一点的了解。

# Node和Zone的概念

我想先讲一下这两个概念之间的区别，或许有了这两者之间的概念，才能够更好的理解内存管理。至少我走过了这么多弯路之后，才发现自己之前的概念是错误的。


**Node**

```
   Memory

                                3G                                   6G
   [           Node0             |                Node1               ]

```

**Zone**
```
   Memory

                 16M                   4G                            6G
   [   ZONE_DMA   |      ZONE_DMA32     |            ZONE_NORMAL      ]
```

两者结合在一起

```
   Memory

                 16M                   4G                            6G
   [   ZONE_DMA   |      ZONE_DMA32     |            ZONE_NORMAL      ]
                                3G
   ^                             ^                                    ^
   |<---      Node0          --->|<---          Node1             --->|
```

让我尝试用两句话来定义Node和Zone
* Node是从内存亲和性出发的定义，看到的表现是地址上的分布，但实际上不是从地址出发的定义。
* Zone是一个从地址大小出发的定义，不论系统上的内存大小是多少，每个zone的空间是一定的。比如ZONE_DMA一定是16M一下的空间。

根据以上的定义其实可以得出，在某些node上的zone是空的。比如你仔细看上面几个图，node1上就不会存在ZONE_DMA的空间。

# 我的一张图

在文章的一开始，我们就看到了一张经典的对内存管理结构的图。对node和zone有了一定理解之后，尝试将这两者的关系，画成图。其中node_data对应的是pg_data_t。

沿用之前6G内存系统的例子，这时内核初始化完的结构如下：

```
   node_data[0]                                                node_data[1]
   +-----------------------------+                             +-----------------------------+        
   |node_id                <---+ |                             |node_id                <---+ |        
   |   (int)                   | |                             |   (int)                   | |        
   +-----------------------------+                             +-----------------------------+    
   |node_zones[MAX_NR_ZONES]   | |    [ZONE_DMA]               |node_zones[MAX_NR_ZONES]   | |    [ZONE_DMA]       
   |   (struct zone)           | |    +---------------+        |   (struct zone)           | |    +---------------+
   |   +-------------------------+    |0              |        |   +-------------------------+    |empty          |
   |   |                       | |    |16M            |        |   |                       | |    |               |
   |   |zone_pgdat         ----+ |    +---------------+        |   |zone_pgdat         ----+ |    +---------------+
   |   |                         |                             |   |                         |        
   |   |                         |    [ZONE_DMA32]             |   |                         |    [ZONE_DMA32]        
   |   |                         |    +---------------+        |   |                         |    +---------------+   
   |   |                         |    |16M            |        |   |                         |    |3G             |   
   |   |                         |    |3G             |        |   |                         |    |4G             |   
   |   |                         |    +---------------+        |   |                         |    +---------------+   
   |   |                         |                             |   |                         |        
   |   |                         |    [ZONE_NORMAL]            |   |                         |    [ZONE_NORMAL]       
   |   |                         |    +---------------+        |   |                         |    +---------------+   
   |   |                         |    |empty          |        |   |                         |    |4G             |   
   |   |                         |    |               |        |   |                         |    |6G             |   
   +---+-------------------------+    +---------------+        +---+-------------------------+    +---------------+
```

图中有几个点值得注意：

* ZONE_DMA和ZONE_DMA32的大小是固定的
* Node0上的ZONE_NORMAL是空的
* Node2上的ZONE_DMA是空的

怎么样，这样看是不是清楚了一点？

## 验证的源代码

来，给你个源代码，这样你自己回去也可以试试。

注： 输出的数值会和理论值有差别

```
diff --git a/mm/page_alloc.c b/mm/page_alloc.c
index 35dcf33..f974112 100644
--- a/mm/page_alloc.c
+++ b/mm/page_alloc.c
@@ -5667,6 +5667,7 @@ static void __meminit calculate_node_totalpages(struct pglist_data *pgdat,
 	unsigned long realtotalpages = 0, totalpages = 0;
 	enum zone_type i;

+	pr_err("Node[%d] Zone info:\n",pgdat->node_id);
 	for (i = 0; i < MAX_NR_ZONES; i++) {
 		struct zone *zone = pgdat->node_zones + i;
 		unsigned long zone_start_pfn, zone_end_pfn;
@@ -5687,6 +5688,9 @@ static void __meminit calculate_node_totalpages(struct pglist_data *pgdat,
 			zone->zone_start_pfn = 0;
 		zone->spanned_pages = size;
 		zone->present_pages = real_size;
+		pr_err("\t Zone[%d] -> [%lx -- %lx]\n",
+			i, (zone_start_pfn),
+			(zone_start_pfn + size));

 		totalpages += size;
 		realtotalpages += real_size;
```

PS: free_area_init()中也有相关的日志输出。

# 划分node/zone的边界

既然每个node都由pgdat表示，那么就需要看看这部分信息是如何初始化的。

函数free_area_init是负责初始化的，这里我们主要观察该函数的行为。而该函数在整个系统初始化过程的位置参见[memblock][2]中的**初始化流程**。

```
free_area_init()
    arch_zone_lowest_possible_pfn[]     // 划分zone的边界
    arch_zone_highest_possible_pfn[]
    find_zone_movable_pfns_for_nodes();

    free_area_init_node(nid)            // 初始化node/zone，设置每个node/zone的范围
        calculate_node_totalpages()     // 计算出node/zone的大小和边界
            zone_spanned_pages_in_node()
                adjust_zone_range_for_zone_movable() // 根据movable zone调整
        free_area_init_core(pgdat)
            zone_init_internals(zone, j, nid, freesize)
            init_currently_empty_zone(zone, zone_start_pfn, size)
                zone_init_free_lists(zone)
                zone->initialized = 1
    node_set_state(nid, N_MEMORY), if pgdat->node_present_pages

    memmap_init()                       // 初始化page struct
        memmap_init_zone_range(zone, start_pfn, end_pfn, &hole_pfn)
            memmap_init_range(, MEMINIT_EARLY, NULL, MIGRATE_MOVABLE)
                __init_single_page(page, pfn, zone, nid)
                __SetPageReserved(page), if context == MEMINIT_HOTPLUG
                set_pageblock_migratetype(page, MIGRATE_MOVABLE)
            init_unavailable_range(*hole_pfn, start_pfn, zone_id, nid)
                __init_single_page(page, pfn, zone, nid)
                __SetPageReserved(page)
```

## 相关日志

内核启动过程中，我们可以通过free_area_init()中输出的日志观察每个node/zone所使用的区域。

日志中搜索的关键字是："Zone ranges:"， "Early memory node ranges"。

运行起来后，能在文件/proc/zoneinfo中看到更多的数据。

# ZONE_MOVABLE

这次花了点时间看了下ZONE_MOVABLE，本来以为这个zone和其他的没有什么区别。实际不然。

这个zone的大小，在x86上，是计算出来的，而不是像其他的zone有明确的事先边界。这里我把和ZONE_MOVABLE计算相关的代码在下面着重标出来。

```
free_area_init()
    arch_zone_lowest_possible_pfn[]
    arch_zone_highest_possible_pfn[]

    find_zone_movable_pfns_for_nodes();              // 根据配置计算每个节点ZONE_MOVABLE的边界
        find_usable_zone_for_movable();              // 选定最后一个有空间的zone作为候选
            movable_zone = zone_index;
        计算每个节点上movable_zone中需要预留的空间   // 就是在选定的movable_zone中，扣一块出来

    // 打印Zone ranges

    // 打印zone_movable_pfn[]

    free_area_init_node(nid)
        calculate_node_totalpages()
            zone_spanned_pages_in_node()             // 计算zone的真实边界
                adjust_zone_range_for_zone_movable() // 根据zone_movable_pfn调整zone的边界
            zone_absent_pages_in_node()              // 计算zone范围中的空缺：包括hole以及movable/normal之间的重叠部分

    memmap_init()
        memmap_init_zone_range()
    ...
```

## 为什么要有ZONE_MOVABLE

如果说其他zone的存在是因为物理访问的限制，比如某些设备只能访问32位以内的内存，所以划分了ZONE_DMA32，那ZONE_MOVABLE则更多是软件上的。

可以看ZONE_MOVABLE的注释：

> ZONE_MOVABLE is similar to ZONE_NORMAL, except that it contains
>  movable pages with few exceptional cases described below. Main use
>  cases for ZONE_MOVABLE are to make memory offlining/unplug more
>  likely to succeed, and to locally limit unmovable allocations

所以ZONE_MOVABLE没有硬性的边界，其边界是人为可以调整，并且在启动时计算出来的。

## 计算ZONE_MOVABLE的边界

默认情况下我们看不到这个zone的存在，有下面几种方式可以开启：

  * movablecore
  * kernelcore

这两者是正反两面，我们只看kernelcore。

使用kernelcore有三种参数方式：

  * kernelcore=512M     # 512MB
  * kernelcore=20%      # 总内存的20%
  * kernelcore=mirror

其大致意思就是希望至少留多少空间给kernel，这部分因为是kernel用，所以我不希望它热插拔。然后剩下的部分呢，拿去做movable就好。

根据这个参数，find_zone_movable_pfns_for_nodes()会计算出对每个numa节点的边界.

  * 如果没有设置参数，或者要求的kernelcore大于所有内存，则不会进行计算
  * movable_zone只会在最后一个zone，find_usable_zone_for_movable
  * 根据kernelcore平均计算出每个节点上预留的大小kernelcore_node.
  * 每个节点上预留kernelcore_node，并计算出zone_movable_pfn[nid]

然后，这个计算出的zone_movable_pfn会在adjust_zone_range_for_zone_movable()时使用。

  * ZONE_MOVABLE: 因为是软件计算出来的，所以zone_start_pfn就是zone_movable_pfn[nid]
                  而zone_end_pfn就是movable_zone的最大值
  * 非ZONE_MOVABLE且!mirrored_kernelcore: 如果当前的这段区域和zone_movable_pfn[nid]
                  重叠，则切分

## 调整ZONE_MOVABLE的边界

很有意思的是，我发现find_zone_movable_pfns_for_nodes()完成后，内核打印Zone ranges和Movable zone start for each node的时候，实际上movable zone的区域还没有落到实处。

因为内核此时输出的日志限制，zone normal还是原来的范围。

```
 Zone ranges:
   DMA      [mem 0x0000000000001000-0x0000000000ffffff]
   DMA32    [mem 0x0000000001000000-0x00000000ffffffff]
   Normal   [mem 0x0000000100000000-0x00000001bfffffff]
   Device   empty
 Movable zone start for each node
   Node 1: 0x0000000100000000
```

真正让zone movalbe出现的，还要依赖函数adjust_zone_range_for_zone_movable()。

```
static void __init adjust_zone_range_for_zone_movable(int nid,
					unsigned long zone_type,
					unsigned long node_end_pfn,
					unsigned long *zone_start_pfn,
					unsigned long *zone_end_pfn)
{
	/* Only adjust if ZONE_MOVABLE is on this node */
	if (zone_movable_pfn[nid]) {
		/* Size ZONE_MOVABLE */
		if (zone_type == ZONE_MOVABLE) {
			*zone_start_pfn = zone_movable_pfn[nid];
			*zone_end_pfn = min(node_end_pfn,
				arch_zone_highest_possible_pfn[movable_zone]);

		/* Adjust for ZONE_MOVABLE starting within this range */
		} else if (!mirrored_kernelcore &&
			*zone_start_pfn < zone_movable_pfn[nid] &&
			*zone_end_pfn > zone_movable_pfn[nid]) {
			*zone_end_pfn = zone_movable_pfn[nid];

		/* Check if this whole range is within ZONE_MOVABLE */
		} else if (*zone_start_pfn >= zone_movable_pfn[nid])
			*zone_start_pfn = *zone_end_pfn;
	}
}
```

到这里，zone movable的区域才算是真的出现了。而被挖掉一块的zone，也剔除了被占用的部分。

## mirrored_kernelcore

之前我们看到kernelcore命令行参数有三种方式，其中最后一种mirror和前两者的处理不同。

当我们指定内核命令行参数kernelcore=mirror后，就会设置mirrored_kernelcore为true。这样在find_zone_movable_pfns_for_nodes()中就会特殊处理。

但是有意思的是，这个命令行选项还要配合memblock_has_mirror()使用。

```
cmdline_parse_kernelcore()
    parse_option_str(p, "mirror")
        mirrored_kernelcore = true

start_kernel()
    setup_arch()
        efi_find_mirror()
            memblock_mark_mirror(start, size)                // 根据硬件属性标记镜像内存区域
                system_has_some_mirror = true;
                memblock_setclr_flag(&memblock.memory, ... , MEMBLOCK_MIRROR)

    mm_core_init_early()
        free_area_init()
            find_zone_movable_pfns_for_nodes
                if (mirrored_kernelcore) {
                    zone_movable_pfn[] 设定为非镜像内存的最小值
		}
```

这里计算zone_movable_pfn时，就变成了取节点上非镜像内存的最小值。

但是这样的话，最后通过adjust_zone_range_for_zone_movable()调整，得到实际的zone边界。我觉得这意味着硬件标识的mirror区域也是在低zone范围内的.

# 思考题

每个page作为链表元素链接在了free_list上，那page数据结构本身放在哪里呢？你猜的到吗？


[1]: https://www.kernel.org/doc/gorman/html/understand/understand005.html
[2]: /mm/02-memblock.md

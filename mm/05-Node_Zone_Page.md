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
    free_area_init_node(nid)            // 初始化node/zone，设置每个node/zone的范围
        calculate_node_totalpages()     // 计算出node/zone的大小和边界
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

# 思考题

每个page作为链表元素链接在了free_list上，那page数据结构本身放在哪里呢？你猜的到吗？


[1]: https://www.kernel.org/doc/gorman/html/understand/understand005.html
[2]: /mm/02-memblock.md
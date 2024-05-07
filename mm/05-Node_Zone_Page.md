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

# 初始化pgdat

既然每个node都由pgdat表示，那么就需要看看这部分信息是如何初始化的。

函数free_area_init是负责初始化的，这里我们主要观察该函数的行为。而该函数在整个系统初始化过程的位置参见[memblock][2]中的**初始化流程**。

```
free_area_init()

```

# Page在哪？

刚才我们已经看到了，内存被划分为node，每个node又划分成了zone。虽然不是每个zone都有内存，但是内核把page相关的信息还是保存在了zone内。

这个逻辑也比较好理解，需要做内存分配的时候，首先我们先选择从哪个node上分配。这样可以找到更适合本地执行的内存。然后再去着想要在哪个zone上去分配。

这个时候你可能会问，为什么不先按照zone去分配呢？有些设备只能使用ZONE_DMA的内存，如果按照ZONE的顺序不是更快一些？嗯是的，内核也提供了这种配置，可以先按照zone的顺序而不是node的顺序。至于是怎么实现的呢？ 这个留给大家自己探索～

好，回到本节。

我们来看看zone和page是怎么联系的。

```
      struct zone
      +------------------------------+      The buddy system
      |free_area[MAX_ORDER]  0...10  |
      |   (struct free_area)         |
      |   +--------------------------+
      |   |nr_free                   |  number of available pages
      |   |(unsigned long)           |  in this zone
      |   |                          |
      |   +--------------------------+
      |   |                          |           free_area[0]
      |   |free_list[MIGRATE_TYPES]  |  Order0   +-----------------------+
      |   |(struct list_head)        |  Pages    |free_list              |
      |   |                          |           |  (struct list_head)   |
      |   |                          |           +-----------------------+
      |   |                          |
      |   |                          |           free_area[1]
      |   |                          |  Order1   +-----------------------+
      |   |                          |  Pages    |free_list              |
      |   |                          |           |  (struct list_head)   |
      |   |                          |           +-----------------------+
      |   |                          |
      |   |                          |              .
      |   |                          |              .
      |   |                          |              .
      |   |                          |
      |   |                          |
      |   |                          |           free_area[10]
      |   |                          |  Order10  +-----------------------+
      |   |                          |  Pages    |free_list              |
      |   |                          |           |  (struct list_head)   |
      |   |                          |           +-----------------------+
      |   |                          |
      +---+--------------------------+
```

概念很简单，但描述起来确实有些困难。

* 每个zone上有一个free_area[]的数组
* 每个数组是一个链表
* 每个链表元素指向一个表达 2^N大小内存的page

嗯，有点绕，希望你能看懂哈～

这其实就是我们通常说的buddy system。分配的时候找到对应的zone，查看是否有可以使用的页面。如果指定大小的页面没有，就去更高阶的链表上找。

好了，这次就先到这里，希望能给到你一点帮助～

# zonelist 分配内存的顺序

其实我有点纠结，这部分内容要放在哪里。放在这，因为正好讲了node和zone的关系。但是内存分配的内容是在下一节涉及，所以放在这里又感觉有点早。

暂且先放在这里吧，以后哪天觉得不合适了，我再来调整。

内存按照Node/Zone划分的目的最终还是要为分配/回收服务。毕竟内存是有限的，所以分配时有一个需求就是按照什么样的顺序从node/zone上去分配内存。zonelist就是这样一个按照zone顺序排好的链表。在分配的时候，如果某个zone找不到可用内存，就按照zonelist上的顺序挨个寻找。

OK，好像一句话就说完了。还是亲眼渐渐zonelist的模样吧。

```
   node_data[]
   +-----------------------------+
   |node_zonelists[MAX_ZONELISTS]|
   |   (struct zonelist)         |
   |   +-------------------------+
   |   |_zonerefs[]              | = MAX_NUMNODES * MAX_NR_ZONES + 1
   |   | (struct zoneref)        | Node 0:
   |   |  +----------------------+    [ZONE_NORMAL]        [ZONE_DMA32]         [ZONE_DMA]
   |   |  |zone                  |    +---------------+    +---------------+    +---------------+
   |   |  |   (struct zone*)     |    |               |    |               |    |               |
   |   |  |zone_idx              |    |               |    |               |    |               |
   |   |  |   (int)              |    +---------------+    +---------------+    +---------------+
   +---+--+----------------------+
                                   Node 1:

                                      [ZONE_NORMAL]        [ZONE_DMA32]         [ZONE_DMA]
                                      +---------------+    +---------------+    +---------------+
                                      |               |    |               |    |               |
                                      |               |    |               |    |               |
                                      +---------------+    +---------------+    +---------------+
```

所以每个node_data上都有自己的zonelist，用来表示在该numa节点上分配内存时如何按照zone找到空闲内存的顺序。

## 如何做到节点按照round robin排序

commit 54d032ced98378bcb9d32dd5e378b7e402b36ad8 描述了之前内核中的一个bug。在多numa节点情况下，zonelist并没有按照round robin的顺序排列，从而导致了内存访问差异。我们来看下原因。

假设有一个4个numa节点的机器，节点之间的distance如下。

```
Node 0  1  2  3
----------------
0    10 12 32 32
1    12 10 32 32
2    32 32 10 12
3    32 32 12 10
```

在改动前，每个节点的zonelist上nume节点顺序如下：

```
Node  Fallback list
---------------------
0     0 1 2 3
1     1 0 3 2
2     2 3 0 1
3     3 2 0 1 <-- Unexpected fallback order
```

可以看到在numa节点2/3上的zonelist顺序是一样的。这样导致在2/3节点上访问远端内存时都先从0上分配。

这样会有两个问题：

  * 软件层面，对0上pgdat/zone/page的访问增加，竞争增加
  * 硬件层面，内存带宽可能打满

所以期望的情况每个节点的zonelist上numa节点顺序如下：

```
Node Fallback list
------------------
0    0 1 2 3
1    1 0 3 2
2    2 3 0 1
3    3 2 1 0
```

可以看出，zonelist的顺序是按照distance排序的，且如果distance相等时，会有round robin的计算。

要做到这点，代码中只做了这么一个改动。

```
-                       node_load[node] = load;
+                       node_load[node] += load;
```

这样就可以把之前找出来过的节点放到列表的后面了。

## 遍历zonelist

既然讲到了zonelist的构建，就顺便看一下遍历吧。

遍历zonelist主要是在__alloc_pages()函数，目的就是为了按照顺序找到有空闲内存的zone进行分配：

  * preferred_zoneref = first_zones_zonelist()
  * for_next_zone_zonelist_nodemask()

## 无锁更新

自从有了内存热插拔后，zonelist就面临着一个问题：

  * 更新zonelist是否需要持锁

因为在分配页的时候需要遍历zonlist，这就意味着如果持锁更新，就会影响到整个系统的性能。

我查看了一下历史更新，发现这个事情还是挺有意思的。下面是相关的commit：

  * 6811378e7d8b 2006-06-23 [PATCH] wait_table and zonelist initializing for memory hotadd: update zonelists
  * 4eaf3f64397c 2010-05-25 mem-hotplug: fix potential race while building zonelist for new populated zone
  * 9d3be21bf9c0 2017-09-06 mm, page_alloc: simplify zonelist initialization
  * 11cd8638c37f 2017-09-06 mm, page_alloc: remove stop_machine from build_all_zonelists
  * b93e0f329e24 2017-09-06 mm, memory_hotplug: get rid of zonelists_mutex

从历史记录来看，最开始是有锁的。到现在其实把锁去掉了。

# 思考题

每个page作为链表元素链接在了free_list上，那page数据结构本身放在哪里呢？你猜的到吗？


[1]: https://www.kernel.org/doc/gorman/html/understand/understand005.html
[2]: /mm/02-memblock.md
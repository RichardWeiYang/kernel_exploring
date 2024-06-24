memblock作为系统启动初期的内存管理系统，在系统启动后就显得不是那么灵活了。在这个时候，内核就提出了页分配器来管理物理内存。而我们在Node->Zone->Page这篇中看到的这个层次结构，就是为了页分配器做的准备。

# 页从哪来

一路走来，我们回顾一下内存信息是怎么收集上来的

```
    BIOS -> e820 -> memblock
```

那现在就要看系统是怎么从memblock的信息中，组合成page的信息的。

众所周知，在内核中我们使用page结构体来表示物理页。在SPARSEMEM这篇中我们也对page结构体的存储位置做了探索。那在最开始的时候，这个结构体是空的。那什么时候系统把物理实际存在的页面和这个page结构体对应上呢？系统是什么时候确认系统到底是有那些page可以使用的呢？如何把page结构体保存到相应的Zone结构体中的呢？

这个过程就在memblock_free_all()函数中。

```c
start_kernel()
    mm_core_init()
        mem_init()
            memblock_free_all()
                free_unused_memmap()                // 释放内存空洞对应的page struct
                reset_all_zones_managed_pages()     // 清除临时的managed_pages
                free_low_memory_core_early()
                    memblock_clear_hotplug(0, -1)
                    memmap_init_reserved_pages()    // set PageReserved
                    __free_memory_core()
                        __free_pages_memory(start_pfn, end_pfn)
                            memblock_free_pages()
```

在这个过程中你可以看到熟悉的for_each_free_mem_range()。对了，这就是遍历memory_block的内存信息，来填充page结构体的过程了。

PS: 当有了defer_init后，大部分的page struct初始化由单独的线程调用deferred_init_memmap()[延迟初始化][4]。

# 页到哪去

刚才我们已经看到了，内存被划分为node，每个node又划分成了zone。虽然不是每个zone都有内存，但是内核把page相关的信息还是保存在了zone内。这样就形成了node-zone-page的关联关系。

这个逻辑也比较好理解，需要做内存分配的时候，首先我们先选择从哪个node上分配。这样可以找到更适合本地执行的内存。然后再去着想要在哪个zone上去分配。

这个时候你可能会问，为什么不先按照zone去分配呢？有些设备只能使用ZONE_DMA的内存，如果按照ZONE的顺序不是更快一些？嗯是的，内核也提供了这种配置，可以先按照zone的顺序而不是node的顺序。至于是怎么实现的呢？ 这个留给大家自己探索～

好，回到本节。

我们来看看node/zone和page是怎么联系的。

```
      struct zone
      +------------------------------+      The buddy system
      |free_area[MAX_ORDER]  0...10  |
      |   (struct free_area)         |
      |   +--------------------------+
      |   |nr_free                   |  number of available pages
      |   |(unsigned long)           |  in this free_area[] list
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
* 每个数组里有MIGRATE_TYPE个链表
* 每个链表元素指向一个表达 2^N 大小内存的page

嗯，有点绕，希望你能看懂哈～

这其实就是我们通常说的buddy system。分配的时候找到对应的zone，查看是否有可以使用的页面。如果指定大小的页面没有，就去更高阶的链表上找。

# 伙伴系统

## 你不是一个人

内核中的buddy system可谓是久仰大名，如雷贯耳。这年头找对象不容易，看看人内核里面是怎么找的，咱也好好学学，取取经～

## 敬个礼，握握手，你是我的好朋友

好了，说正经的，人是怎么找的呢？

```c

__free_one_page()
...
	while (order < max_order - 1) {
		buddy_idx = __find_buddy_index(page_idx, order);
		buddy = page + (buddy_idx - page_idx);
		if (!page_is_buddy(page, buddy, order))
			goto done_merging;
		/*
		 * Our buddy is free or it is CONFIG_DEBUG_PAGEALLOC guard page,
		 * merge with it and move up one order.
		 */
		if (page_is_guard(buddy)) {
			clear_page_guard(zone, buddy, order, migratetype);
		} else {
			list_del(&buddy->lru);
			zone->free_area[order].nr_free--;
			rmv_page_order(buddy);
		}
		combined_idx = buddy_idx & page_idx;
		page = page + (combined_idx - page_idx);
		page_idx = combined_idx;
		order++;
	}
...
```

其中重要的就是__find_buddy_index()。
```c
/*
 * Locate the struct page for both the matching buddy in our
 * pair (buddy1) and the combined O(n+1) page they form (page).
 *
 * 1) Any buddy B1 will have an order O twin B2 which satisfies
 * the following equation:
 *     B2 = B1 ^ (1 << O)
 * For example, if the starting buddy (buddy2) is #8 its order
 * 1 buddy is #10:
 *     B2 = 8 ^ (1 << 1) = 8 ^ 2 = 10
 *
 * 2) Any buddy B will have an order O+1 parent P which
 * satisfies the following equation:
 *     P = B & ~(1 << O)
 *
 * Assumption: *_mem_map is contiguous at least up to MAX_ORDER
 */
static inline unsigned long
__find_buddy_index(unsigned long page_idx, unsigned int order)
{
	return page_idx ^ (1 << order);
}
```

人注释写得真好，还有个例子给你看。我这个人有个毛病，一旦看懂了代码就老兴奋了。忍不住多说两句，估计人真正的大牛对这种代码了如指掌，根本没兴趣了。

我个人看完这个代码有两点体会：
* 使用某个位置上的数做异或就像是一个开关，原来有的就清，没的就置位
*  两个小伙伴的上一级伙伴，是这个位为零的那个

## 举个栗子

     +--------+--------+--------+--------+
     |8       |9       |10      |11      |
     +--------+--------+--------+--------+

     [======  2  =====] [=====  2  ======]

     [###############  4  ###############]

举个栗子， 从上面这个图看出，Order为1的时候，（8，9）为一组，（10，11）为一组。

 ```
          8   ^ 2 =  0b 1000 ^ 0b 0010 = 0b 1010 = 10
          10 ^ 2 =  0b 1010 ^ 0b 0010 = 0b 1000 = 8
 ```
可以看到，异或的结果就是order 2的那一位去反。很有意思。恩，是不是可以理解为男生找女生，女生找男生呢？ （如果你硬要找一样的，我就没办法了）

那我们再来看如何找上一级的伙伴。从图上直观来看， (8,9) (9,10)的上一级就是数量为4的组。那他的首页面index就是8。

好了看看代码实现的效果
 ```
          8   & ~2 =  0b 1000 & 0b 1101 = 0b 1000 = 8
          10 & ~2 =  0b 1010 & 0b 1101 = 0b 1000 = 8
 ```
直观的效果就是大家都将order 2的那一位清除。哇哦，so easy.

大家注意一下实际代码中的一句

```c
		combined_idx = buddy_idx & page_idx;
		page_idx = combined_idx;
```

在循环中，page_idx将作为下一次循环要去寻找伙伴的索引。那你想到了啥？
对了，正是因为自己的上一级伙伴是把order 2的那一位清空，所以自己和自己小伙伴做与运算就得到了上一级～

好了，我想你知道内核页分配中伙伴是怎么找的了～

到这里，就到这里，休息休息~

# 分配内存的顺序--zonelist

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

## 构造zonelist的代码在哪里

虽然构造zonelist的代码不复杂，不过我们还是看看这部分是在哪里做的。

```
mm_core_init()
    build_all_zonelists(NULL)
        build_all_zonelists_init()
            __build_all_zonelists(NULL)
                write_seqlock_irqsave(&zonelist_update_seq, flags);
                build_zonelists(pgdat);
                    build_zonelists_in_node_order(pgdat, node_order, nr_nodes);
                    build_thisnode_zonelists(pgdat);
                write_sequnlock_irqrestore(&zonelist_update_seq, flags);
```

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

# 释放

通常我们都是先讲分配，再讲释放的。但是这次要到过来。为什么呢，因为在free_all_bootmem()中就是调用了内存释放的方法来初始化page结构的。所以还是先来看看释放吧。

我们通常使用的函数是free_pages()。但是最后都调用到了函数__free_one_page()。这个函数其实比较简单，就是算的东西稍微有点绕。我们都知道页分配器中使用的是伙伴系统，也就是相邻的两个页是一对儿。所以在释放的时候，会去判断自己的伙伴是不是也是空闲的。如果是的话，那就做合并。且继续判断。

关于伙伴查找的算法，后面详细讲。有兴趣的可以仔细看。

那最后释放到哪里了呢？

对了，就是zone->free_area[order].free_list上了。

## 各种释放接口

内核中有很多释放内存的地方，这里我们总结一下。

* __free_pages_core，这是将内存从memblock/hotplug时，加入到buddy system
* 核心 __free_one_page，最后都要通过这个函数将内存放到free_list上
* free_one_page 这个和__free_one_page的差别就是拿了zone的锁，大部分实际会走这里
* __free_pages_ok/free_unref_page/free_unref_folio, 在free_one_page前调用free_pages_prepare()
* __free_pages/put_page/folio_put, 先减去引用计数，再调用free_unref_[page|folio]

### __free_pages_core

这是个比较特殊的接口，从memblock/hotplug来的内存通过这个接口释放到buddy system。

### __free_one_page流程

这个是最内存的释放函数了，基本所有的page都是靠他释放到free_list上的。

```c
__free_one_page(page, pfn, zone, order, migratetype, fpi_flags)
    __mod_zone_freepage_state(zone, 1 << order, migratetype)      // 增加NR_FREE_PAGES计数

    buddy = find_buddy_page_pfn(page, pfn, order, &buddy_pfn)
        __buddy_pfn = __find_buddy_pfn(pfn, order)
            return page_pfn ^ (1 << order)
        buddy = page + (__buddy_pfn - pfn)
        page_is_buddy(page, buddy, order)
    del_page_from_free_list(buddy, zone, order)                   // 如果buddy刚好空闲，先把他从free_list上摘下来
    combined_pfn = buddy_pfn & pfn
    page = page + (combined_pfn - pfn)
    pfn = combined_pfn
    order++

done_merging:
    set_buddy_order(page, order)
        set_page_private(page, order)
        __SetPageBuddy(page)

    add_to_free_list[_tail](page, zone, order, migratetype)       // 添加到free_list上
```

### free_one_page

这个和__free_one_page的差别在与持锁操作

### __free_pages_ok/free_unref_[page|foio]

在free_one_page前，调用了free_pages_prepare()

### __free_pages/put_page

这两个都是先减去引用计数，再调用free_unref_page。
这两者有细微的差别，具体可以看__free_pages的注释。

# 分配

页分配的核心函数是__alloc_pages_noprof()。正如函数的注释上写的"This is the 'heart' of the zoned buddy allocator."。

在[Node->Zone->Page][2]中我们已经看到了，内存被分为了node，zone来管理。这么管理的目的也就是为了在分配的时候，能能够找到合适的内存。所以在分配的时候，就是按照优先级搜索node和zone，如果找到匹配的zone则在该zone的free_area链表中取下一个。

这里面涉及到两个问题：

* 按照什么顺序从zone上分配内存
* 怎么判断应该要去下一个zone上去找

第一个问题我们前面看到过了，就是zonelist。而第二个问题是由[水线][5]来判断的。这涉及到了内存回收，已经是另一个课题了。

和释放内存对称，分配的时候也可能会分配到高阶的page。如果发生这种情况，则会将高阶部分内存拆分到低阶的page，再放回到对应的free_area中。这部分代码可以看__rmqueue_smallest()。

# 引用计数

随着伙伴系统越来越复杂，就会需要一些辅助信息来帮助管理。比如记录当前页面有多少使用者。在页结构体中有两个重要的引用计数：

  * _refcount
  * _mapcount

前者用来记录该页面当前的使用者个数，而后者标示当前页面映射到页表的次数。

而且在内核中如果前者的数目大于后者，则称这个页面“pin”住了。

## page->_refcount

这个值用来表示有多少人在使用这个page，当page在buddy分配器中_refcount为0。

```
memmap_init()
    memmap_init_zone()
        init_page_count()
            set_page_count(page, 1)    <--- set to 1

mem_init()
    memblock_free_all()
        free_low_memory_core_early()
            __free_memory_core()
                ...
                __free_pages_core()
                    set_page_refcounted()   <--- set to 1 again
                    __free_pages()
```

之后有新的使用者，一般会用get_page来表示，防止错误释放页面。

同样在put_page中，就会检查_refcount是不是为零。如果减到零，就会把页面归还给buddy分配器。

值得注意的是如果是compound page那么这个引用计数会加在head page上。

## page->_mapcount

用来表示映射到用户空间的次数。从-1开始。所以在分配页面时需要确认这点。

```
check_new_page()
  if (unlikely(atomic_read(&page->_mapcount) != -1))
    return false;
```

不过要获得一个页面被映射到进程的次数，就稍微复杂了点。这里涉及到了两个函数：

  * page_mapcount
  * total_mapcount

其中page_mapcount是针对一个PTE page来看的。
而total_mapcount则是针对一个THP来看的。

里面的原因在[THP和mapcount之间的恩恩怨怨][3]一文中做了详细的阐述。


[1]: http://blog.csdn.net/richardysteven/article/details/52332040
[2]: /mm/05-Node_Zone_Page.md
[3]: /virtual_mm/02-thp_mapcount.md
[4]: /mm/54-defer_init.md
[5]: /mm_reclaim/02-watermark.md

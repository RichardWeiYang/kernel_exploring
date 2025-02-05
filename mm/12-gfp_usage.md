前文中我们了解了页分配的核心函数__alloc_frozen_pages_noprof()在分配页的过程中，受到四个参数的影响。

  * gfp
  * order
  * preferred_nid
  * nodemask

这里我们对第一个参数gfp展开看看。

该类型的定义在include/linux/gfp.h文件中，不过每个bit的定义在include/linux/gfp_types.h文件中。

# 对zone的挑选

前文中我们已经看到，内核将物理内存按照node/zone的方式进行了划分。

分配时，究竟从那个node/zone分配则受到了内核中几个部分的影响：

  * zonelist
  * gfp_zone

其中zonelist是系统层面的，而gfn则可以由用户调整。


## node_zonelists

node_zonelists是pgdat中的一个成员，乍一看有点复杂，但实际上并不难。我先画一个图给大家看看。

```
    +-----------------------------+
    |node_zonelists[MAX_ZONELISTS]|
    |   (struct zonelist)         |
    |   +-------------------------+
    |   |_zonerefs[]              | = MAX_NUMNODES * MAX_NR_ZONES + 1
    |   | (struct zoneref)        | Node 0:
    |   |  +----------------------+
    |   |  |zone                  |    [ZONE_NORMAL]        [ZONE_DMA32]         [ZONE_DMA]
    |   |  |   (struct zone*)     |    +---------------+    +---------------+    +---------------+
    |   |  |zone_idx              |    |               |    |               |    |               |
    |   |  |   (int)              |    |               |    |               |    |               |
    |   |  |                      |    +---------------+    +---------------+    +---------------+
    |   |  |                      |
    |   |  |                      | Node 1:
    |   |  |                      |
    |   |  |                      |    [ZONE_NORMAL]        [ZONE_DMA32]         [ZONE_DMA]
    |   |  |                      |    +---------------+    +---------------+    +---------------+
    |   |  |                      |    |               |    |               |    |               |
    |   |  |                      |    |               |    |               |    |               |
    |   |  |                      |    +---------------+    +---------------+    +---------------+
    |   |  |                      |
    +---+--+----------------------+
```

这张图呢其实画得不是很好，结构上有点不是很准确。我来说明一下：

  * node_zonelists是一个数组，不过这个数组只有两个元素：ZONELIST_FALLBACK 和 ZONELIST_NOFALLBACK
  * node_zonelists每一个元素的类型是zonelist，这也是一个数组，长度是(MAX_NUMNODES * MAX_NR_ZONES + 1)
  * zonelist的每个元素的类型是zoneref，其中就两个元素：zone, zone_idx

最后总结一下， zonelist就是一个排序后的数组，按照numa distance将分配过程中fallback的顺序按照node/zone的顺序排列好。这样如果在最合适的node/zone上没有足够的内存，就依次找下一个。

这个zonelist构造的函数是build_zonelists。而寻找的顺序地方在get_page_from_freelist()的for_next_zone_zonelist_nodemask()体现。

## gfp_zone()

上面我们看到了从zone中分配内存的一个顺序，但是我们是怎么去指定最合适的zone的呢？那就是gfp_zone()这个函数了。在prepare_alloc_pages函数中就调用了gfp_zone()从gfp中解析处最合适的zone。

内核开发者将zone的信息巧妙得编码在了gfp中，就让我们来看看这个编码解码的过程吧。

### 四位zone的标示

GFP是get free page的缩写，在分配内存时通常都要指定一个GFP来标示你希望得到什么样的内存。然后page allocator会尽量满足你。

GFP是一个非常复杂的位图组合，这里我们主要关心和zone相关的位。

```
#define ___GFP_DMA             0x01u
#define ___GFP_HIGHMEM         0x02u
#define ___GFP_DMA32           0x04u
#define ___GFP_MOVABLE         0x08u

#define __GFP_DMA      ((__force gfp_t)___GFP_DMA)
#define __GFP_HIGHMEM  ((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_DMA32    ((__force gfp_t)___GFP_DMA32)
#define __GFP_MOVABLE  ((__force gfp_t)___GFP_MOVABLE)  /* ZONE_MOVABLE allowed */
#define GFP_ZONEMASK   (__GFP_DMA|__GFP_HIGHMEM|__GFP_DMA32|__GFP_MOVABLE)
```

也就是GFP中一共有四位(低四位)来标示想要从哪个zone分配内存。

### GFP_ZONE_TABLE

好了，接下来就是内核开发者玩花样的时候了。他们将16个可能出现的zone组合编码进了GFP_ZONE_TABLE。
为什么是16个呢？因为 2 ^ 4 = 16。

但是低三位中，只能同时有一位为1，所以这16中组合中我们可以看到多个BAD。

也就是这么一张看上去像表一样的东西。

```
bit       result
=================
0x0    => NORMAL
0x1    => DMA or NORMAL
0x2    => HIGHMEM or NORMAL
0x3    => BAD (DMA+HIGHMEM)
0x4    => DMA32 or NORMAL
0x5    => BAD (DMA+DMA32)
0x6    => BAD (HIGHMEM+DMA32)
0x7    => BAD (HIGHMEM+DMA32+DMA)
0x8    => NORMAL (MOVABLE+0)
0x9    => DMA or NORMAL (MOVABLE+DMA)
0xa    => MOVABLE (Movable is valid only if HIGHMEM is set too)
0xb    => BAD (MOVABLE+HIGHMEM+DMA)
0xc    => DMA32 or NORMAL (MOVABLE+DMA32)
0xd    => BAD (MOVABLE+DMA32+DMA)
0xe    => BAD (MOVABLE+DMA32+HIGHMEM)
0xf    => BAD (MOVABLE+DMA32+HIGHMEM+DMA)
```

然后，是的，还有个然后。内核大师们将这16种情况硬是写到了GFP_ZONE_TABLE中。

```
#define GFP_ZONES_SHIFT ZONES_SHIFT

#if 16 * GFP_ZONES_SHIFT > BITS_PER_LONG
#error GFP_ZONES_SHIFT too large to create GFP_ZONE_TABLE integer
#endif

#define GFP_ZONE_TABLE ( \
	(ZONE_NORMAL << 0 * GFP_ZONES_SHIFT)				       \
	| (OPT_ZONE_DMA << ___GFP_DMA * GFP_ZONES_SHIFT)		       \
	| (OPT_ZONE_HIGHMEM << ___GFP_HIGHMEM * GFP_ZONES_SHIFT)	       \
	| (OPT_ZONE_DMA32 << ___GFP_DMA32 * GFP_ZONES_SHIFT)		       \
	| (ZONE_NORMAL << ___GFP_MOVABLE * GFP_ZONES_SHIFT)		       \
	| (OPT_ZONE_DMA << (___GFP_MOVABLE | ___GFP_DMA) * GFP_ZONES_SHIFT)    \
	| (ZONE_MOVABLE << (___GFP_MOVABLE | ___GFP_HIGHMEM) * GFP_ZONES_SHIFT)\
	| (OPT_ZONE_DMA32 << (___GFP_MOVABLE | ___GFP_DMA32) * GFP_ZONES_SHIFT)\
)
```

为了防止GFP_ZONE_TABLE超出ulong的大小，还专门做了个判断。

那最后怎么从GFP上取出对应的zone呢？这个过程在gfp_zone()中实现。其实就是将GFP_ZONE_TABLE右移再取低位，就得到了这张表中对应的zone的数值了。

内核开发这还是挺会玩的。

对应的还有个位图叫GFP_ZONE_BAD，用来判断GFP中zone相关位是否合法。这个就留给大家去探究了。

# migrate type的选择

除了指定node和zone，分配过程中还能指定pageblock的migratetype。

还记得free_are[NR_PAGE_ORDERS].free_list[MIGRATE_TYPES]这个数组么？在每个order的链表里，又分成了MIGRATE_TYPES个链表。而究竟在哪个链表上去分配，gfp中也可以指定。

PS: gfp指定migrate_type的能力好像是有限的，并不能选择到所有的可能性。

## gfp_migratetype()

上面我们看到gfp的低四位用来表示zone，有意思的是用来表示migratetype的是GFP[3:4]。也就是第三位被重复利用了。

一共只有2bit，所以一共可能的组合是4种：

  * 0                                     -> MIGRATE_UNMOVABLE
  * ___GFP_MOVABLE                        -> MIGRATE_MOVABLE
  * ___GFP_RECLAIMABLE                    -> MIGRATE_RECLAIMABLE
  * ___GFP_MOVABLE | ___GFP_RECLAIMABLE   -> MIGRATE_HIGHATOMIC

这个逻辑都在gfp_migratetype()里，不过这里面有好多个BUILD_BUG_ON()，对里面的要求还是很严格的。

前文中我们了解了内核如何将内存按照numa，地址空间做了划分。那么在分配的时候我们如何能够指定从哪一个node，哪个zone去分配呢？

这个过程受到了内核中几个部分的影响：

  * zonelist
  * gfp_zone
  * gfp_migratetype

其中zonelist是系统层面的，而gfn则可以由用户调整，和zonelist配合发挥作用。

# node_zonelists

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

# gfp_zone

上面我们看到了从zone中分配内存的一个顺序，但是我们是怎么去指定最合适的zone的呢？那就是gfp_zone()这个函数了。在prepare_alloc_pages函数中就调用了gfp_zone()从gfp中解析处最合适的zone。

内核开发者将zone的信息巧妙得编码在了gfp中，就让我们来看看这个编码解码的过程吧。

## 四位zone的标示

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

也就是GFP中一共有四位来标示想要从哪个zone分配内存。

## GFP_ZONE_TABLE

好了，接下来就是内核开发者玩花样的时候了。他们将16个可能出现的zone组合编码进了GFP_ZONE_TABLE。
为什么是16个呢？因为 2 ^ 4 = 16。

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

那最后怎么从GFP上取出对应的zone呢？这个过程在gfp_zone()中实现。其实就是将GFP_ZONE_TABLE右移再取低位，就得到了这张表中对应的zone的数值了。

内核开发这还是挺会玩的。

对应的还有个位图叫GFP_ZONE_BAD，用来判断GFP中zone相关位是否合法。这个就留给大家去探究了。

# gfp_migratetype

除了指定node和zone，分配过程中还能指定pageblock的migratetype。当然暂时我还不知道这个migratetype到底是有什么用，不过还不妨碍我们去研究它。

上面我们看到gfp的低四位用来表示zone，有意思的是用来表示migratetype的是GFP[3:4]。也就是第三位被重复利用了。但是理论上migratetype不止4中组合，但是为什么可以用两位来表达呢？

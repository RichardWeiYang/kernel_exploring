上问中我们已经看到linux内核将内存划分为node/zone，也看到了伙伴系统上如何取出和放回一个页。

那么问题是我们如何在分配时指定希望从哪个node哪个zone上来分配内存呢？如果这一点没发做到，好像这一切就没有什么意义了。

答案当然是有的了。

其实规定某个node很简单，也用不着我来解释。而如何指定zone就有点意思了，主要还是内核开发者玩了一点小花样。
这一切都藏在了GFP中，让我们来瞧瞧吧。

# 四位zone的标示

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

# GFP_ZONE_TABLE

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

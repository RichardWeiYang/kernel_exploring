在page allocator中有两个概念会用到：

  * pageblock
  * migratetype


在这里我们好好研究一下。

# 相关数据结构

这两个概念有相关性，开始我以为两个是一个意思，但仔细看了后发现是两个不同的概念。

## pageblock

pageblock的定义是在mem_section中。

```
     mem_section
     +-----------------------------+
     |section_mem_map              |
     |   (unsigned long)           |
     +-----------------------------+
     |usage                        |
     |   (mem_section_usage *)     |
     |   +-------------------------+
     |   |subsection_map           |
     |   |                         |
     .   .                         .
     |   |                         |    [(1UL << (PFN_SECTION_SHIFT - pageblock_order))] 个
     |   |                         |    each 4/8bits represents a pageblock type
     |   |pageblock_flags[0]       |    get_pfnblock_flags_mask() / get_pfnblock_migratetype()
     |   |    (unsigned long)   ---|--->+----+----+----+---...---+----+----+
     +---+-------------------------+    |    |    |    |         |    |    |
                                        |    |    |    |         |    |    |
                                        +----+----+----+---...---+----+----+
```

可以看到，pageblock是粒度比在section小的一个概念。（不过从大小上看，不一定比buddy
allocator中的MAX_ORDER要大，参见isolate_single_pageblock的注释）

再看一眼，实际上整个mem_section_usage定义的时候是一个指针，所以相关的存储空间是在初始化时，动态分派的。sparse_init_nid()。
其中pageblock的空间是个另一个usage中的成员一起分配的，sparse_usage_init()。其中计算mem_section_usage大小的函数是mem_section_usage_size()。

```
static unsigned long usemap_size(void)
{
	return BITS_TO_LONGS(SECTION_BLOCKFLAGS_BITS) * sizeof(unsigned long);
}

size_t mem_section_usage_size(void)
{
	return sizeof(struct mem_section_usage) + usemap_size();
}
```

所以，实际上给pageblock分配的空间是通过usemap_size()计算出来的。可以看出，pageblock是以bitmap的形式出现的。那就再看看这个bitmap是怎么定义的。


```
#define PB_migratetype_bits 3
/* Bit indices that affect a whole block of pages */
enum pageblock_bits {
	PB_migrate,
	PB_migrate_end = PB_migrate + PB_migratetype_bits - 1,
			/* 3 bits required for migrate types */
	PB_compact_skip,/* If set the block is skipped by compaction */

#ifdef CONFIG_MEMORY_ISOLATION
	/*
	 * Pageblock isolation is represented with a separate bit, so that
	 * the migratetype of a block is not overwritten by isolation.
	 */
	PB_migrate_isolate, /* If set the block is isolated */
#endif
	/*
	 * Assume the bits will always align on a word. If this assumption
	 * changes then get/set pageblock needs updating.
	 */
	__NR_PAGEBLOCK_BITS
};

#define NR_PAGEBLOCK_BITS (roundup_pow_of_two(__NR_PAGEBLOCK_BITS))
#define SECTION_BLOCKFLAGS_BITS \
	((1UL << (PFN_SECTION_SHIFT - pageblock_order)) * NR_PAGEBLOCK_BITS)
```

SECTION_BLOCKFLAGS_BITS分成两部分，前者是计算一个section中可以划分多少pageblock，后者是给出一个pageblock需要多少bit来表示属性。

NR_PAGEBLOCK_BITS是根据pageblock_bits的定义来的

  * 如果没有定义CONFIG_MEMORY_ISOLATION，得出的结果是4
  * 如果定义了CONFIG_MEMORY_ISOLATION，得出的结果是8

## migratetype

migratetype和buddy allocator的关系更紧密一些。

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
      |   |                          |              .
      |   |                          |              .
      |   |                          |              .
      |   |                          |
      |   |                          |           free_area[10]
      |   |                          |  Order10  +-----------------------+
      |   |                          |  Pages    |free_list              |
      |   |                          |           |  (struct list_head)   |
      |   |                          |           +-----------------------+
      +---+--------------------------+
```

通常我们都说分配内存时，是从free_area[].free_list上找，但是往往忽略的是free_list其实是按照migratetype做了区分的。

所以在分配的时候我们可以指定从哪种类型分配，也可以将特定的内存放到指定的类型上。这样就不会被其他人影响。

```
enum migratetype {
	MIGRATE_UNMOVABLE,
	MIGRATE_MOVABLE,
	MIGRATE_RECLAIMABLE,
	MIGRATE_PCPTYPES,	/* the number of types on the pcp lists */
	MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
#ifdef CONFIG_CMA
	/*
	 * MIGRATE_CMA migration type is designed to mimic the way
	 * ZONE_MOVABLE works.  Only movable pages can be allocated
	 * from MIGRATE_CMA pageblocks and page allocator never
	 * implicitly change migration type of MIGRATE_CMA pageblock.
	 *
	 * The way to use it is to change migratetype of a range of
	 * pageblocks to MIGRATE_CMA which can be done by
	 * __free_pageblock_cma() function.
	 */
	MIGRATE_CMA,
	__MIGRATE_TYPE_END = MIGRATE_CMA,
#else
	__MIGRATE_TYPE_END = MIGRATE_HIGHATOMIC,
#endif
#ifdef CONFIG_MEMORY_ISOLATION
	MIGRATE_ISOLATE,	/* can't allocate from here */
#endif
	MIGRATE_TYPES
};
```

比如说，我们可以看到在pageblock_bits和migratetype定义中，都有一个CONFIG_MEMORY_ISOLATION的定义。而这里又都定义了一个isolate的类型。

这个我估计和内存的隔离功能是密切相关的。

# 相关操作

看过了数据结构，接下来看看有哪些常用的操作。

## pageblock

对pageblock来说，有两组四个个

  * __set_pfnblock_flags_mask(, MIGRATETYPE_AND_ISO_MASK)
  * __get_pfnblock_flags_mask()
  * set_pfnblock_bit()
  * clear_pfnblock_bit()

其实就是找到对应pfn在mem_section->usage->pageblock_flags中的位置，然后读取/设置对应的pageblock值。也就是那4位或者8位。或者是只处理其中某一位。

不过其中有意思的是__set_pfnblock_flags_mask()最后一个参数，MIGRATETYPE_AND_ISO_MASK。这个我们留到下面再说。

## migratetype

对migratetype来说，有三组七个（怎么这么不对称）

  * get_pfnblock_migratetype()
  * set_pageblock_migratetype()
  * get_pageblock_migratetype()
  * set_pageblock_isolate()
  * clear_pageblock_isolate()
  * get_pageblock_isolate()
  * toggle_pageblock_isolate()

从这个名字上可以看出，isolate这个属性是有点特别的，为它单独出了四个函数。而且好像和migratetype中其他的属性有点格格不入。

后面四个函数都是围绕pageblock中PB_migrate_isolate这一个bit来操作的。而前面两个则是整个pageblock的操作。然而要看清楚关键，还是要看前两者的实现。

比如在[get|set]_pageblock_migratetype()，都单独处理了isolate。甚至想要设置MIGRATE_ISOLATE，必须要用set_pageblock_isolate()。

这样做的原因是因为这个commit

```
ommit 42f46ed99ac6c07adf7f3bcbe9040b0c52d62d0f
Author: Zi Yan <ziy@nvidia.com>
Date:   Mon Jun 16 22:11:09 2025 -0400

    mm/page_alloc: pageblock flags functions clean up
    
    Patch series "Make MIGRATE_ISOLATE a standalone bit", v10.
```

把MIGRATE_ISOLATE单独列出来了。

# pageblock和migratetype的关系

pageblock和migratetype之间有着密切的联系，但是两者又不一样。让我们合起来看看。

我们先从一个定义开始，还记得我们前面提到过的MIGRATETYPE_AND_ISO_MASK吗？ 这里就要请他出场了。

```
#define MIGRATETYPE_AND_ISO_MASK \
	(((1UL << (PB_migrate_end + 1)) - 1) | BIT(PB_migrate_isolate))
```

这个mask就是所有有效的migratetype位。后面一个isolate的bit一眼看出来了，前半部分有点意思。但是结合pageblock_bits的定义就看出来了。**3 bits required for migrate types**。

好了，这也就是告诉我们pageblock中，低三位是用来存储/表达migratetype的。下面这么对比一看，是不是就清楚一些了？

```
enum pageblock_bits {                                                          enum migratetype {
	PB_migrate,                                                            	MIGRATE_UNMOVABLE,
	PB_migrate_end = PB_migrate + PB_migratetype_bits - 1,                 	MIGRATE_MOVABLE,
			/* 3 bits required for migrate types */                	MIGRATE_RECLAIMABLE,
	PB_compact_skip,/* If set the block is skipped by compaction */        	MIGRATE_PCPTYPES,	/* the number of types on the pcp lists */
                                                                               	MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
#ifdef CONFIG_MEMORY_ISOLATION                                                 #ifdef CONFIG_CMA
	/*                                                                     	/*
	 * Pageblock isolation is represented with a separate bit, so that     	 * MIGRATE_CMA migration type is designed to mimic the way
	 * the migratetype of a block is not overwritten by isolation.         	 * ZONE_MOVABLE works.  Only movable pages can be allocated
	 */                                                                    	 * from MIGRATE_CMA pageblocks and page allocator never
	PB_migrate_isolate, /* If set the block is isolated */                 	 * implicitly change migration type of MIGRATE_CMA pageblock.
#endif                                                                         	 *
	/*                                                                     	 * The way to use it is to change migratetype of a range of
	 * Assume the bits will always align on a word. If this assumption     	 * pageblocks to MIGRATE_CMA which can be done by
	 * changes then get/set pageblock needs updating.                      	 * __free_pageblock_cma() function.
	 */                                                                    	 */
	__NR_PAGEBLOCK_BITS                                                    	MIGRATE_CMA,
};                                                                             	__MIGRATE_TYPE_END = MIGRATE_CMA,
                                                                               #else
                                                                               	__MIGRATE_TYPE_END = MIGRATE_HIGHATOMIC,
                                                                               #endif
                                                                               #ifdef CONFIG_MEMORY_ISOLATION
                                                                               	MIGRATE_ISOLATE,	/* can't allocate from here */
                                                                               #endif
                                                                               	MIGRATE_TYPES
                                                                               };
```

另外，为了防止后面有人误操作，在migratetype中新增加类型导致pageblock中放不下，还在get_pfnblock_bitmap_bitidx()中增加了编译时报错.

```
	BUILD_BUG_ON(__MIGRATE_TYPE_END >= (1 << PB_migratetype_bits));
```

这样保证了migratetype不会超出pageblock中规定的范围。

# 启动时，内存放在那个类型上

前面我们在migratetype部分看到free_list[]其实是一个数组，根据pageblock中的值，会把page存放到对应的链表。

那我们现在来看看系统启动时，会把page放到那个链表。

## 设置migratetype

启动过程中，我们把对应的pageblock设置成了MIGRATE_MOVABLE。

```
memmap_init()
    memmap_init_zone_range()
        memmap_init_range( MIGRATE_MOVABLE, false)
            init_pageblock_migratetype( MIGRATE_MOVABLE, false)
```

## 释放过程

启动过程中，__free_memory_core()会将memblock中记录的内存释放到buddy。跳过前面的部分，我们看看到最后的时候是怎么处理的。

```
free_one_page(zone, page, pfn, order, )
    split_large_buddy()                                       // 如果order > pageblock_order, 则会切分后继续
        mt = get_pfnblock_migratetype(page, pfn)              // 得到对应的migratetype
        __free_one_page(page, pfn, zone, order, mt)
            __add_to_free_list(page, zone, order, mt, )       // 最后存放到指定的order，指定的migratetype上
```

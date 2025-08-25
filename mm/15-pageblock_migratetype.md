在page allocator中有两个概念会用到：

  * pageblock
  * migratetype


在这里我们好好研究一下。

# 相关定义

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
     |   |                         |    [(1UL << (PFN_SECTION_SHIFT - pageblock_order))]
     |   |                         |    each 4bits represents a pageblock type
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
#define NR_PAGEBLOCK_BITS (roundup_pow_of_two(__NR_PAGEBLOCK_BITS))
#define SECTION_BLOCKFLAGS_BITS \
	((1UL << (PFN_SECTION_SHIFT - pageblock_order)) * NR_PAGEBLOCK_BITS)
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
```

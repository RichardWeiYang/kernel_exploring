今天要来讲讲linux内核中的SPARSEMEM。

这其实是一个内核可配置项，比如在arch/x86/Kconfig中可以看到如下的配置。

```
config ARCH_SPARSEMEM_ENABLE
	def_bool y
	depends on X86_64 || NUMA || X86_32 || X86_32_NON_STANDARD
	select SPARSEMEM_STATIC if X86_32
	select SPARSEMEM_VMEMMAP_ENABLE if X86_64
```

也就是只要是满足depends on这四个条件之一，默认都会开启SPARSEMEM。

书接上回，我们提出了一个思考题：page结构体究竟是存放在哪里？你有找到么？如果没有的话，那就由我来给大家絮叨絮叨。另外前文那张经典的数据结构关系图中一个未露面的成员zone_mem_map在本文中将会出现。不过刚才那张图有点老，当前代码中使用的可能是他的孙子辈吧。

估计你现在还是一头雾水，莫名来了这么多名词，概念，也不知道哪个是哪个。其实这个东西说难也不难。刚开始理解时略有些麻烦，但是看明白之后也没有太多内容。

找个舒服的地儿，倒杯茶，让我给你一一道来。

# 有啥好处?

先来说说用它的好处，否则大家都不知道这是干啥的。在原始[补丁][1]中，作者做了些比较详细的描述。我列举一下：

* 对于有内存空洞的设备，减少系统struct page使用的内存
* 内存热插拔系统需要使用
* 在NUMA系统上支持内存的overlap

后两者究竟是怎么用的，暂时还不清楚。减少系统struct page使用的内存，那是肯定的。因为在最原始的实现中，系统struct page是一个超大的静态数组。不管对应的物理地址是否有内存存在，都会有对应的struct page对应。而使用了SPARSEMEM后，只有有效的page才会有对应的struct page存在。

在减少内存使用的同时，也带来了一个问题。

* pfn_to_page()计算会比原来的慢

针对这点，内核中又引入了SPARSEMEM_VMEMMAP。这个以后再来讲解，这次先关注SPARSEMEM本身。

# 实现思想

SPARSEMEM提出了一个新的概念

> section

这个section就是比page更大一些的内存区域，但是呢一般意义上又比node的范围要小些。比如在x86_64平台上，这个的大小就是(2^27) = 128M。

整个系统的物理内存就被分成一个个section，并由mem_section结构体表示。而这个结构体中保存了该section范围内struct page结构体的地址。

那是怎么做到节省了struct page的内存的呢？

在没有使用SPARSEMEM前，page结构体是一个连续数组。也就是说如果整个系统最大会支持多少内存，都得预先分配好空间。哪怕中间有一段物理内存是空的，也得留着。

使用了SPARSEMEM后，如果有一个section是空的，也就是内存中有128M的空洞，那这个section中的struct page内存空间就不会被分配。从而节省了一定的空间。这么看其实要达到节省内存的条件还是蛮高的，至少得有128M的空洞。等等，而且这128M还得要对齐的。这种情况真的多么？

而且这也是为什么做内存热插拔时，要求至少时128M的原因。有意思吧。

# 简单图解

讲的多了，用图来给大家展示一下。

先来看看全局mem_section数组。这个数组的每一个元素是mem_section结构体，名字上有重复，大家要注意。

每个数组的元素管理了128M的内存。这样，可以很容易的通过pfn（page frame number）找到对应的section。

```
    mem_section[NR_SECTION_ROOTS][SECTIONS_PER_ROOT]

    = [DIV_ROUND_UP(NR_MEM_SECTIONS, SECTIONS_PER_ROOT)] [SECTIONS_PER_ROOT]

        [0]          [1]                                [SECTIONS_PER_ROOT - 1]
        +------------+------------+        +------------+------------+
    [0] |            |            |   ...  |            |            |
        +------------+------------+        +------------+------------+

        +------------+------------+        +------------+------------+
    [1] |            |            |   ...  |            |            |
        +------------+------------+        +------------+------------+

        +------------+------------+        +------------+------------+
    [2] |            |            |   ...  |            |            |
        +------------+------------+        +------------+------------+
```

怎么样，这么看是不是还挺简单的。就是用数组去覆盖整个内存的区域，然后分块管理么。

接下来我们再来看看section中是如何管理这个struct page的。

```
    mem_section                     
    +-----------------------------+
    |pageblock_flags              |
    |   (unsigned long *)         |
    |                             |
    |                             |
    +-----------------------------+         mem_map[PAGES_PER_SECTION]
    |section_mem_map              |  ---->  +------------------------+
    |   (unsigned long)           |    [0]  |struct page             |
    |                             |         |                        |
    |                             |         +------------------------+
    +-----------------------------+    [1]  |struct page             |
                                            |                        |
                                            +------------------------+
                                       [2]  |struct page             |
                                            |                        |
                                            +------------------------+
                                            |                        |
                                            .                        .
                                            .                        .
                                            .                        .
                                            |                        |
                                            +------------------------+
                                            |struct page             |
                                            |                        |
                                            +------------------------+
                   [PAGES_PER_SECTION - 1]  |struct page             |
                                            |                        |
                                            +------------------------+
```

需要注意一点的是section_mem_map对应的内存是动态分配的。所以如果这个section对应的物理内存不存在，即便是mem_section的空间会被分配，其对应的struct page的空间是不会被分配的。这样起到了节省内存的作用。

**page结构体在这里**
**page结构体在这里**
**page结构体在这里**

重要的事情说三遍。

# 变慢了么？

刚才我们就说到了使用了这个模型后pfn_to_page()的访问会变慢。那我们来看看是不是真的呢？

先看看原始的定义：

```
#define __pfn_to_page(pfn)	(mem_map + ((pfn) - ARCH_PFN_OFFSET))
```

再来看看更改后的定义：

```
#define __pfn_to_page(pfn)				\
({	unsigned long __pfn = (pfn);			\
	struct mem_section *__sec = __pfn_to_section(__pfn);	\
	__section_mem_map_addr(__sec) + __pfn;		\
})
```

更改后，需要先找到section，然后再计算出page的地址。这个样比原来多了一步，确实是慢了一些。

# 互相转换

理解了SPARSEMEM的模型后，我们就可以理解这两个在内核中重要的转换了

* __pfn_to_page()
* __page_to_pfn()

也就是物理地址和page结构体之间的互相转换。

```
/*
 * Note: section's mem_map is encoded to reflect its start_pfn.
 * section[i].section_mem_map == mem_map's address - start_pfn;
 */
#define __page_to_pfn(pg)					\
({	const struct page *__pg = (pg);				\
	int __sec = page_to_section(__pg);			\
	(unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec)));	\
})

#define __pfn_to_page(pfn)				\
({	unsigned long __pfn = (pfn);			\
	struct mem_section *__sec = __pfn_to_section(__pfn);	\
	__section_mem_map_addr(__sec) + __pfn;		\
})
```

它们都定义在同一个头文件中。这个定义也非常的巧妙，section_mem_map尽然包含了start_pfn。这样的结果就是计算时不需要再去计算一次整个section的起始地址了。是不是觉得很赞～

好了，这个SPARSEMEM就讲到这里～我想大家已经知道page结构体是在什么地方的了～

[1]: https://lwn.net/Articles/134804/

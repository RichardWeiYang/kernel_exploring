# 虚拟地址和物理地址

## __pa_symbol()

获取内核程序中符号的物理地址，也就是nm vmlinux命令能显示出的内容。

比如 nm vmlinux | grep -w _text，其结果是“ffffffff81000000 T _text”。

```
#define __START_KERNEL_map	_AC(0xffffffff80000000, UL)


#define __pa_symbol(x) \
	__phys_addr_symbol(__phys_reloc_hide((unsigned long)(x)))


#define __phys_addr_symbol(x) \
	((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

这里我们只看x86_64的定义。从定义中我们可以看到，符号的物理地址是虚拟地址 减去 __START_KERNEL_map 再加上 phys_base。

当配置了kaslr时，内核加载的物理地址和编译时的加载地址会有个偏移，这个偏移记录在phys_base。具体解释可见[常用全局变量][1]

正因为phys_base的偏移是基于__START_KERNEL_map计算出来的，所以如此计算后就得到了内核代码中符号的物理地址。

## __pa()

计算出虚拟地址对应的物理地址。

```
static __always_inline unsigned long __phys_addr_nodebug(unsigned long x)
{
	unsigned long y = x - __START_KERNEL_map;

	/* use the carry flag to determine if x was < __START_KERNEL_map */
	x = y + ((x > y) ? phys_base : (__START_KERNEL_map - PAGE_OFFSET));

	return x;
}

#define __phys_addr(x)		__phys_addr_nodebug(x)

#define __pa(x)		__phys_addr((unsigned long)(x))
```

其实这里计算的时候分了两种情况。

* 虚拟地址 > __START_KERNEL_map
* 其他虚拟地址

第一种情况，说明虚拟地址在内核代码空间，所以实际上就退化成和__pa_symbol()一样。
第二种情况，转换过程和__va相反。这说明传入的虚拟地址需要是在内核页表上映射的空间内。

## __va() / phys_to_virt()

只能对有内核映射的地址调用该函数，来获得对应地址的虚拟地址。

```
static inline void *phys_to_virt(phys_addr_t address)
{
	return __va(address);
}

#define __PAGE_OFFSET_BASE_L4	_AC(0xffff888000000000, UL)
unsigned long page_offset_base __ro_after_init = __PAGE_OFFSET_BASE_L4;

#define __PAGE_OFFSET           page_offset_base
#define PAGE_OFFSET		((unsigned long)__PAGE_OFFSET)

#define __va(x)			((void *)((unsigned long)(x)+PAGE_OFFSET))
```

再x86_64架构下，大概率是这么个定义。

也就是一个物理地址（非内核代码范围内的）的虚拟地址 = 虚拟地址 + 0xffff888000000000。

这是因为在映射整个内存空间时，是这么操作的。具体参见[映射完整物理地址][2]

# pfn和struct page

这里先列出只有sparsemem的情况

## __pfn_to_page(pfn)

```
static inline struct page *__section_mem_map_addr(struct mem_section *section)
{
	unsigned long map = section->section_mem_map;
	map &= SECTION_MAP_MASK;
	return (struct page *)map;
}

#define __pfn_to_page(pfn)				\
({	unsigned long __pfn = (pfn);			\
	struct mem_section *__sec = __pfn_to_section(__pfn);	\
	__section_mem_map_addr(__sec) + __pfn;		\
})
```

先根据pfn找到对应的section，然后对section->section_mem_map做一个偏移运算得到struct page的地址。

## __page_to_pfn(pg)

```
static inline unsigned long page_to_section(const struct page *page)
{
	return (page->flags >> SECTIONS_PGSHIFT) & SECTIONS_MASK;
}

#define __page_to_pfn(pg)					\
({	const struct page *__pg = (pg);				\
	int __sec = page_to_section(__pg);			\
	(unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec)));	\
})
```

先根据page找到对应的section，注意这里找是因为把sectino number写在了page->flags里。
然后再对section->section_mem_map做一个偏移得到pfn。

# 虚拟地址和struct page

有了上面两个转换，自然可以推导出这个转换。也就是形成了地址和struct page之间的关系。

## virt_to_page()

```
#define virt_to_page(kaddr)	pfn_to_page(__pa(kaddr) >> PAGE_SHIFT)
```

这个转换没有什么太多可以解释的，不过这里有另一个检查的函数值得一看。

```
bool __virt_addr_valid(unsigned long x)
{
	unsigned long y = x - __START_KERNEL_map;

	/* use the carry flag to determine if x was < __START_KERNEL_map */
	if (unlikely(x > y)) {
		x = y + phys_base;

		if (y >= KERNEL_IMAGE_SIZE)
			return false;
	} else {
		x = y + (__START_KERNEL_map - PAGE_OFFSET);

		/* carry flag will be set if starting x was >= PAGE_OFFSET */
		if ((x > y) || !phys_addr_valid(x))
			return false;
	}

	return pfn_valid(x >> PAGE_SHIFT);
}
```

就是这个用来判断虚拟地址是否有效的函数。可以看出在内核中认为有效的虚拟地址空间有两个：

* __START_KERNEL_map以上的内核代码空间
* PAGE_OFFSET以上的direct mapping空间

其余部分都是无效空间。

## page_to_virt()

```
#define page_to_virt(x)	__va(PFN_PHYS(page_to_pfn(x)))
```

[1]: /mm/common/00_global_variable.md
[2]: /kernel_pagetable/04-map_whole_memory.md
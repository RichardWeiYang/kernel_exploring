# __pa_symbol()

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

# __pa()

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

# __va() / phys_to_virt()

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



[1]: /mm/common/00_global_variable.md
[2]: /kernel_pagetable/04-map_whole_memory.md
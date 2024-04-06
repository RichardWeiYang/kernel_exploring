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


# __va()


# __pa()

# phys_to_virt()

[1]: /mm/common/00_global_variable.md
# max_pfn

最大page frame number.

```
max_pfn = e820__end_of_ram_pfn();
```

# MAX_PHYSMEM_BITS

最大支持物理内存

```
# define MAX_PHYSMEM_BITS	(pgtable_l5_enabled() ? 52 : 46)
```

也就是说，没有5级页表的情况下，物理内存最多是64T。

# phys_base

定义在head_64.S

```
SYM_DATA(phys_base, .quad 0x0)
EXPORT_SYMBOL(phys_base)
```

赋值在__startup_64()

```
	/*
	 * Compute the delta between the address I am compiled to run at
	 * and the address I am actually running at.
	 */
	load_delta = physaddr - (unsigned long)(_text - __START_KERNEL_map);
	RIP_REL_REF(phys_base) = load_delta;
```

在配置了kaslr时，内核加载地址会和编译时的地址有个偏移。phys_base就记录了这个偏移。（所以看上去这个变量名字好像不是很准确。）

在后续fixup页表，以及计算符号物理地址时(__pa_symbol)会用到。
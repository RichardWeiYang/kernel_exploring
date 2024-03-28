# __pa_symbol

符号的物理地址

```
#define __pa_symbol(x) \
	__phys_addr_symbol(__phys_reloc_hide((unsigned long)(x)))


#define __phys_addr_symbol(x) \
	((unsigned long)(x) - __START_KERNEL_map + phys_base)
```

当配置了kaslr时，内核加载的物理地址和编译时的加载地址会有个偏移，这个偏移记录在phys_base。


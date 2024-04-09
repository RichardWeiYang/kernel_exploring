刚才我们看到，页表中映射了内核代码区域。那如果内核要访问其他的区域呢？内核又能够访问多大的内存空间呢？现在我们就去探索一下。

# 找到代码

据我所知，相关的秘密就在这个函数之中了。

```
init_mem_mapping()
	/* the ISA range is always mapped regardless of memory holes */
	init_memory_mapping(0, ISA_END_ADDRESS);

	...

	/*
	 * If the allocation is in bottom-up direction, we setup direct mapping
	 * in bottom-up, otherwise we setup direct mapping in top-down.
	 */
	if (memblock_bottom_up()) {
		unsigned long kernel_end = __pa_symbol(_end);

		/*
		 * we need two separate calls here. This is because we want to
		 * allocate page tables above the kernel. So we first map
		 * [kernel_end, end) to make memory above the kernel be mapped
		 * as soon as possible. And then use page tables allocated above
		 * the kernel to map [ISA_END_ADDRESS, kernel_end).
		 */
		memory_map_bottom_up(kernel_end, end);
		memory_map_bottom_up(ISA_END_ADDRESS, kernel_end);
	} else {
		memory_map_top_down(ISA_END_ADDRESS, end);
	}
```

上面这两个函数都最后调用kernel_physical_mapping_init()来实际把映射关系写入了页表。

# 做个实验

具体代码分析就不在这看了。做个小实验，看一下映射后的效果。

```
 arch/x86/mm/init.c | 37 +++++++++++++++++++++++++++++++++++++
 1 file changed, 37 insertions(+)

diff --git a/arch/x86/mm/init.c b/arch/x86/mm/init.c
index 679893ea5e68..f9d46ee0b4f9 100644
--- a/arch/x86/mm/init.c
+++ b/arch/x86/mm/init.c
@@ -754,6 +754,27 @@ static void __init init_trampoline(void)
 void __init init_mem_mapping(void)
 {
 	unsigned long end;
+	int i, j;
+	pud_t *pud;
+
+	pr_err(": level3_kernel_pgt = %lx\n", __pa_symbol(level3_kernel_pgt));
+	for (i = 0; i < 512; i++) {
+		if (pgd_val(early_top_pgt[i])) {
+			pr_err(": early_top_pgt[%d] 0x%lx\n",
+				i, pgd_val(early_top_pgt[i]));
+
+			pud = (pud_t *)pgd_page_vaddr(early_top_pgt[i]);
+
+			for(j = 0; j < 512; j++)
+				if (pud_val(pud[j]))
+					pr_err(": \t pud[%d] = %lx\n",
+						j, pud_val(pud[j]));
+                }
+        }
+	for (i = 0; i < 512; i++)
+		if (pgd_val(init_top_pgt[i]))
+			pr_err(": init_top_pgt[%d] 0x%lx\n",
+				i, pgd_val(init_top_pgt[i]));
 
 	pti_check_boottime_disable();
 	probe_page_size_mask();
@@ -791,6 +812,22 @@ void __init init_mem_mapping(void)
 		memory_map_top_down(ISA_END_ADDRESS, end);
 	}
 
+	pr_err(": after memory mapped\n");
+	for (i = 0; i < 512; i++) {
+		if (pgd_val(init_top_pgt[i])) {
+			pr_err(": init_top_pgt[%d] 0x%lx\n",
+				i, pgd_val(init_top_pgt[i]));
+
+			pud = (pud_t *)pgd_page_vaddr(init_top_pgt[i]);
+
+			for(j = 0; j < 512; j++)
+				if (pud_val(pud[j]))
+					pr_err(": \t pud[%d] = %lx\n",
+						j, pud_val(pud[j]));
+		}
+	}
+
+
 #ifdef CONFIG_X86_64
 	if (max_pfn > max_low_pfn) {
 		/* can we preserve max_low_pfn ?*/
-- 
2.34.1

```

该代码就是打印了两层的页表，显示的是物理地址。实验的结果是：

```
[    0.000000] : init_level3_pgt = 1e0a000
[    0.000000] : init_level4_pgt[511] 0x1e0a067
[    0.000000] : after memory mapped
[    0.000000] : init_level4_pgt[272] 0x2230067
[    0.000000] : 	 pud[0] = 2231067
[    0.000000] : 	 pud[1] = 11ffff067
[    0.000000] : 	 pud[2] = 11fffe067
[    0.000000] : 	 pud[3] = 11fffd067
[    0.000000] : 	 pud[4] = 2233067
[    0.000000] : init_level4_pgt[511] 0x1e0a067
[    0.000000] : 	 pud[510] = 1e0b063
[    0.000000] : 	 pud[511] = 1e0c067
```

首先我们验证了init_level4_pgt的最后一项确实是init_level3_pgt。其次我们看到，映射前后页表的差异。映射后多了第272项，一共有五个表项。

# 增加内存

刚才虚拟机配置的内存是4G，现在我们配置成6G看看。

```
[    0.000000] : after memory mapped
[    0.000000] : init_level4_pgt[272] 0x2230067
[    0.000000] : 	 pud[0] = 2231067
[    0.000000] : 	 pud[1] = 1bcfff067
[    0.000000] : 	 pud[2] = 1bcffe067
[    0.000000] : 	 pud[3] = 1bcffd067
[    0.000000] : 	 pud[4] = 2234067
[    0.000000] : 	 pud[5] = 2235067
[    0.000000] : 	 pud[6] = 2233067
[    0.000000] : init_level4_pgt[511] 0x1e0a067
[    0.000000] : 	 pud[510] = 1e0b063
[    0.000000] : 	 pud[511] = 1e0c067
```

怎么样，是不是多了两项？

对了，现在配置的页表，每个pud映射的是1G，所以你增加了2G就增加了两个表项了。

嗯，我知道你在问为什么原来4G的时候是5个而不是4个表项？ 你猜？

# 映射关系

和之前一样，我们再来计算一下页表的映射关系。

```
    (272) << 39 | (0) << 30
=   8800 0000 0000
```

这说明了虚拟地址0xFFFF 8800 0000 0000和物理地址0x0是对应的。

再来看一下这个定义

```
/*
 * Set __PAGE_OFFSET to the most negative possible address +
 * PGDIR_SIZE*16 (pgd slot 272).  The gap is to allow a space for a
 * hypervisor to fit.  Choosing 16 slots here is arbitrary, but it's
 * what Xen requires.
 */
#define __PAGE_OFFSET_BASE      _AC(0xffff880000000000, UL)

#define __PAGE_OFFSET           __PAGE_OFFSET_BASE
```

有了这两个概念再来物理地址转换虚拟地址的函数

```
#define __va(x)			((void *)((unsigned long)(x)+PAGE_OFFSET))
```

怎么样，是不是这下能看懂了？当然了，代码中本来就是这么写的。我只是把结果先展示给大家，再来反推。

# 不是如来

想当年齐天大圣一个跟斗十万八千里都没有跳出如来佛祖的掌心，所以我也不知道佛祖他的手掌究竟是有多大。不过我们的页表覆盖的内存大小倒是有限制的。

首先地址空间是有限的，现在地址是用64bit表示，所以怎么着也不会超过这个范围。

其次我们在64bit中之用了48bit，至少从4.9的内核看是这样的。

然后我们在看，__PAGE_OFFSET的定义，一下子又把2^48劈成了两半，剩下的还不到一半了。

之前我就在想，那超过了怎么办，会不会把内核的映射空间给冲掉？后来发现自己想多了，人有这么一个定义

```
# define max_physmem_bits	46
```

也就是最大只能是2^46物理内存，所以这剩下来的小于2^47空间还是够的～

思考：这个变量是在哪里被使用到从而保证不超过2^46大小？

其实呢，内核中还有一个描述内存布局的文档可以证明这一点。Documentation/x86/x86_64/mm.txt

```
Virtual memory map with 4 level page tables:

0000000000000000 - 00007fffffffffff (=47 bits) user space, different per mm
hole caused by [48:63] sign extension
ffff800000000000 - ffff87ffffffffff (=43 bits) guard hole, reserved for hypervisor
ffff880000000000 - ffffc7ffffffffff (=64 TB) direct mapping of all phys. memory
ffffc80000000000 - ffffc8ffffffffff (=40 bits) hole
ffffc90000000000 - ffffe8ffffffffff (=45 bits) vmalloc/ioremap space
ffffe90000000000 - ffffe9ffffffffff (=40 bits) hole
ffffea0000000000 - ffffeaffffffffff (=40 bits) virtual memory map (1TB)
... unused hole ...
ffffec0000000000 - fffffbffffffffff (=44 bits) kasan shadow memory (16TB)
... unused hole ...
ffffff0000000000 - ffffff7fffffffff (=39 bits) %esp fixup stacks
... unused hole ...
ffffffef00000000 - fffffffeffffffff (=64 GB) EFI region mapping space
... unused hole ...
ffffffff80000000 - ffffffff9fffffff (=512 MB)  kernel text mapping, from phys 0
ffffffffa0000000 - ffffffffff5fffff (=1526 MB) module mapping space
ffffffffff600000 - ffffffffffdfffff (=8 MB) vsyscalls
ffffffffffe00000 - ffffffffffffffff (=2 MB) unused hole
```

虽然其中还有不少不是很了解，不过可以看到中间有一块64 TB的空间用来直接映射物理内存。而64 TB == 2^ 46。这就是linux内核在64位上现在能够映射的最大物理内存空间了。

PS：感觉快不够用了啊

# 页表容貌

来看看映射完物理地址后的页表吧。

![这里写图片描述](/kernel_pagetable/map_whole_memory.png)

这次最关键的变化就是init_level4_pgt[272]这一项以及之后的内容。在这里映射了系统的整个物理地址。

细心的朋友可能已经注意到了，这次改写的是init_level4_pgt，而不是之前的early_level4_pgt了。

# 内核文档

另外内核文档中也有非常详尽的[页表映射关系][1]。

好，可以休息一下了～

[1]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/x86/x86_64/mm.rst

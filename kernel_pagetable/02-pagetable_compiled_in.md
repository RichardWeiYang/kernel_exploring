刚才的那张页表实在是太粗糙了，到了真正的内核，怎么也得换个漂亮点的页表。

这次的页表就在arch/x86/kernel/head_64.S里面咯～

# 从cr3开始

在x86平台上保存页表起始地址的就是这个cr3寄存器了，那就先看看这个寄存器变成了谁呗。

```
	movq	$(early_level4_pgt - __START_KERNEL_map), %rax

	/* Setup early boot stage 4 level pagetables. */
	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

嗯，这个东西稍微有点绕。phys_base是一个偏移，我们暂且认为就是0吧。所以这次加载到cr3中的地址是early_level4_pgt - __START_KERNEL_map。这里稍微解释一下下，如果实在理解不了，暂时先跳过。

加载到cr3的是一个物理地址，而我们编译出的内核vmlinux是一个ELF文件且最终会在虚拟地址空间运行，所以所有的符号都保存的是虚拟地址。上面这两个动作就是得到early_level4_pgt这个符号在运行时的物理地址。

不管怎么样知道加载到cr3的地址是指向early_level4_pgt的就好～

# early_level4_pgt的容貌

刚才最简单的页表也有三层了，内核中的页表也不例外。


**early_level4_pgt:**

```
NEXT_PAGE(early_level4_pgt)
	.fill	511,8,0
	.quad	level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE
```

**level3_kernel_pgt:**

```
NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	/* (2^48-(2*1024*1024*1024)-((2^39)*511))/(2^30) = 510 */
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE
```

**level2_kernel_pgt:**

```
NEXT_PAGE(level2_kernel_pgt)
	/*
	 * 512 MB kernel mapping. We spend a full page on this pagetable
	 * anyway.
	 *
	 * The kernel code+data+bss must not be bigger than that.
	 *
	 * (NOTE: at +512MB starts the module area, see MODULES_VADDR.
	 *  If you want to increase this then increase MODULES_VADDR
	 *  too.)
	 */
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)
```

# 看图说话

是不是又看得头晕了？ 嗯，不着急，再来看一张图～

![这里写图片描述](/kernel_pagetable/pagetable_compiled.png)

所以这其实是一张编译时就写好的页表。这样看，是不是简单了些？

# 映射关系

细心的朋友可能发现了，这张页表和之前的页表最后一个层级差不多，就是少了点。而最大的差别是第一层上我们使用了最后一个表项，而前一个页表中的第一层使用的是最后一个表项。

对了，我想你猜到了。这就是虚拟地址和物理地址的映射了。

那我们来算一下这次映射关系究竟是什么样子的。

```
    (511) << 39 | (510) << 30
=   FFFF 8000 0000
```

再来看一下变量__START_KERNEL_map的定义

```
#define __START_KERNEL_map	_AC(0xffffffff80000000, UL)
```

x86上只使用了48bit虚拟地址空间，而不是64bit。所以这两个地址就是等价的。经过计算证实了这次映射的就是内核代码空间的页表。You get it?

# 启用虚拟地址

正如之前看到的，虽然使用了页表，但是页表中的物理地址和虚拟地址是一模一样的。经过了这次页表的改造，就有了真正的地址转换了。但是这个时候我们还是运行在一一对应的地址映射空间，还需要跳一次才能够进入真正的虚拟地址映射的空间。

代码很有意思

```
	/* Ensure I am executing from virtual addresses */
	movq	$1f, %rax
	jmp	*%rax
1:
```

就是取到下一个lable的地址，然后跳过去～

好了，内核就这么欢快的在它自己的**虚拟地址**上运行了～

# 后记

写完之后自己再看一遍，却担心在手机前的你并不能真的理解内核页表究竟是如何长成这样的。有些东西不通过自己亲身阅读，加上动手实验，别人再怎么讲也没有办法真的get到那个点。

才发现文字和图片是如此的单薄，我只能告诉你发现的最后结果，却无法帮助你体会探索过程中想通页表模样时那种被电击一般的体验。不知道你是否能够体会，如果此文能对你有一点点的帮助，我也感到非常开心。

加油～

# 更新：2024.03.15

看了下最新的内核，这部分的页表初始化有些许变化。

虽然也是在编译时候就基本写好了页表的内容，但是至少有两点变化：

* 支持5级页表的改动
* 支持内核随机加载

总体还好，只要理解了静态的问题不大。这部分代码在函数__startup_64中完成。

另外时隔多年后回过头来再看，这个页表的最大作用是切换到内核的虚拟地址。也算是兜兜转转又找了一圈。
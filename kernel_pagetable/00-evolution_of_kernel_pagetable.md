内存管理重要的组成部分是虚拟地址和物理地址之间的映射关系。从物理设备上看这部分的功能由页表(page table)实现。

页表本身是存储在物理内存中的一块内存空间，操作系统按照硬件规范填写相应地址形成树状结构。而硬件则根据这个树状结构实现虚拟地址到物理地址的映射。

Intel的手册上写的非常详细，我就截取一张图做示例。

![这里写图片描述](/kernel_pagetable/intel_virtual_phys_add.png)

每个进程都有自己的页表，虽然用户空间的映射依赖进程自身，但所有进程都共享同样的内核页表空间。所以探究内核页表结构是加深内核页表运行机制的方式之一。

然而你知道么，内核页表并非一蹴而就，而是经过了几个步骤才最终成为我们想要的样子。就好像你得先飞升上仙，才能够飞升上神。这是一个道理。

内核页表经过了下面几个变化过程。

内核刚加载时，还是压缩后的状态。这时候内核就有一张非常简陋的页表。直接映射了4G空间。

[未解压时的内核页表][1]

![这里写图片描述](/kernel_pagetable/pagetable_before_decompression.png)

内核解压缩完之后，就换掉了那张简陋的页表。这张页表在编译时就已经有了大概的雏形。不过只映射了内核空间。这时候虚拟机地址已经和物理地址不一样了。

[内核早期的页表][2]

![这里写图片描述](/kernel_pagetable/pagetable_compiled.png)

而出于某些原因，内核只留下了_text到_brk_end这段空间的映射。其余的映射都清零了。



[cleanup_highmap之后的页表][3]

![这里写图片描述](/kernel_pagetable/pagetable_after_cleanup_highmap.png)

只映射了内核的空间肯定是不够的。接着内核就映射了系统上的所有物理地址。

[映射完整物理地址][4]

![这里写图片描述](/kernel_pagetable/map_whole_memory.png)

所有的准备工作做完，最后有切换了一次。从early_level4_pgt切换到了init_level4_pgt。对了，这个就是那个init进程的空间了。

[启用init_level4_pgt][5]

这个图和上面的是一样的，没有变化。关键变化在cr3的内容，而不是页表本身。

内核页表是否还会增正和变化还有待进一步探索，有了我再来改～

[1]: /kernel_pagetable/01-pagetable_before_decompressed.md
[2]: /kernel_pagetable/02-pagetable_compiled_in.md
[3]: /kernel_pagetable/03-pagetable_after_cleanup_highmap.md
[4]: /kernel_pagetable/04-map_whole_memory.md
[5]: /kernel_pagetable/05-switch_to_init_level4_pgt.md

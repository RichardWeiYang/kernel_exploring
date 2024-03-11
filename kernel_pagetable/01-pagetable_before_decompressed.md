这或许是x86平台启动过程中第一张页表了。

之前我们也学习了内核启动镜像bzImage由两部分组成setup.bin和vmlinux.bin。而这张第一章页表就在vmlinux.bin的head.S中。

如果对上述两个文件编译过程不熟悉的话可以参考下面的链接：

* [启动镜像bzImage的前世今生][1]
* [真假vmlinux--vmlinux.bin揭开的秘密][2]

# 先来看个代码

这第一张页表初始化的代码就在arch/x86/boot/compressed/head_64.S中。

```
 /*
  * Build early 4G boot pagetable
  */
	/* Initialize Page tables to 0 */
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
	rep	stosl

	/* Build Level 4 */
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)

	/* Build Level 3 */
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:	movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b

	/* Build Level 2 */
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:	movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b

	/* Enable the boot page tables */
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

看到最后把地址保存到了cr3了么？Yep, you get it.

干这么看确实有点枯燥，不过简单来说就是分别填写了三层结构，构造出了一张覆盖4G大小的页表。

PS: 此时页表并没有启用，而是要等到后面CR0中的PG被置位后。

# 再来看一张图

看着图再去对照代码，我相信你就可以看懂了。

![这里写图片描述](/kernel_pagetable/pagetable_before_decompression.png)

这样是不是清晰了很多。

代码我就不多说了，多看几遍自然就懂了。正所谓

> 代码虐我千百遍，我待代码如初恋

# 重要提醒

大家看这张表，有没有意识到什么特别的地方？对了，虚拟地址和物理地址是一样的。

那内核中的虚拟地址又是如何访问的呢？别着急，这一切才刚刚开始～

[1]: /brief_tutorial_on_kbuild/07_rules_for_bzImage.md
[2]: /brief_tutorial_on_kbuild/09_rule_for_vmlinux_bin.md

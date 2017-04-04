在x86平台的setup_arch中，会对内核的虚拟机地址空间做一个剪切。具体原因可以看代码的注释。

```
/*
 * The head.S code sets up the kernel high mapping:
 *
 *   from __START_KERNEL_map to __START_KERNEL_map + size (== _end-_text)
 *
 * phys_base holds the negative offset to the kernel, which is added
 * to the compile time generated pmds. This results in invalid pmds up
 * to the point where we hit the physaddr 0 mapping.
 *
 * We limit the mappings to the region from _text to _brk_end.  _brk_end
 * is rounded up to the 2MB boundary. This catches the invalid pmds as
 * well, as they are located before _text:
 */
```

说的有点多，最后的结果比较简单，就是只留下了_text到_brk_end之间的页表映射。

那咱们来打印一下，看看效果呗。

# 调试补丁

```
diff --git a/arch/x86/mm/init_64.c b/arch/x86/mm/init_64.c
index 14b9dd7..2ffb7f2 100644
--- a/arch/x86/mm/init_64.c
+++ b/arch/x86/mm/init_64.c
@@ -310,6 +310,25 @@ void __init cleanup_highmap(void)
 	unsigned long vaddr_end = __START_KERNEL_map + KERNEL_IMAGE_SIZE;
 	unsigned long end = roundup((unsigned long)_brk_end, PMD_SIZE) - 1;
 	pmd_t *pmd = level2_kernel_pgt;
+	int i = 0;
+
+	pr_err(": phys_base                %lx\n", phys_base         );
+	pr_err(": #level2_kernel_pgt       %lu\n", KERNEL_IMAGE_SIZE/PMD_SIZE);
+	pr_err(": __START_KERNEL_map       %lx\n", __START_KERNEL_map);
+	pr_err(": __START_KERNEL           %lx\n", __START_KERNEL);
+	pr_err(": _text                    %lx\n", (unsigned long)_text);
+	pr_err(": _brk_end                 %lx\n", (unsigned long)_brk_end);
+	pr_err(": _end                     %lx\n", (unsigned long)_end);
+	pr_err(": __START_KERNEL_map + KI  %lx\n",
+			__START_KERNEL_map + KERNEL_IMAGE_SIZE);
+
+	for (i = 0; i < 512; i++) {
+		if (pmd_none(*(pmd + i)))
+			continue;
+
+		pr_err(": level2_kernel_pgt[%d] = %lx\n",
+			i, pmd_val(*(pmd+i)));
+	}

 	/*
 	 * Native path, max_pfn_mapped is not set yet.
@@ -325,6 +344,17 @@ void __init cleanup_highmap(void)
 		if (vaddr < (unsigned long) _text || vaddr > end)
 			set_pmd(pmd, __pmd(0));
 	}
+
+	pr_err(": 2nd round\n");
+	pmd = level2_kernel_pgt;
+	for (i = 0; i < 512; i++) {
+		if (pmd_none(*(pmd + i)))
+			continue;
+
+		pr_err(": level2_kernel_pgt[%d] = %lx\n",
+			i, pmd_val(*(pmd+i)));
+	}
+
 }

 /*
```

稍微有点长，但是功能却比较简单。

* 打印了几个比较重要的变量
* 打印了level2_kernel_pgt中的非空项

# 几个重要的变量

下面打印了内核中几个比较重要的虚拟地址的值，按照大小排序：

```
[    0.000000] : phys_base                0
[    0.000000] : #level2_kernel_pgt       256
[    0.000000] : __START_KERNEL_map       ffffffff80000000
[    0.000000] : __START_KERNEL           ffffffff81000000
[    0.000000] : _text                    ffffffff81000000
[    0.000000] : _brk_end                 ffffffff82236000
[    0.000000] : _end                     ffffffff82256000
[    0.000000] : __START_KERNEL_map + KI  ffffffffa0000000
```

注，这个内核我disable了RANDOMIZE_MEMORY。

在没有随机放置内核的情况下 phys_base为0，所以内核的起始地址并没有变化和编译时的一样。

这里要看的是那个KI

```
KI = 0xa0000000 - 0x80000000
   = 0x20000000
   = 512MB
```

这个值就是内核镜像的大小，也就是我们需要做内核地址映射的大小。因为每个PMD映射空间是2MB，所以整个内核地址空间需要映射256个entry。

# level2_kernel_pgt的变化

截取调试打印的部分

```
[    0.000000] : level2_kernel_pgt[0] = 1e3
[    0.000000] : level2_kernel_pgt[1] = 2001e3
[    0.000000] : level2_kernel_pgt[2] = 4001e3
[    0.000000] : level2_kernel_pgt[3] = 6001e3
[    0.000000] : level2_kernel_pgt[4] = 8001e3
[    0.000000] : level2_kernel_pgt[5] = a001e3
...
[    0.000000] : level2_kernel_pgt[251] = 1f6001e3
[    0.000000] : level2_kernel_pgt[252] = 1f8001e3
[    0.000000] : level2_kernel_pgt[253] = 1fa001e3
[    0.000000] : level2_kernel_pgt[254] = 1fc001e3
[    0.000000] : level2_kernel_pgt[255] = 1fe001e3

[    0.000000] : 2nd round
[    0.000000] : level2_kernel_pgt[8] = 10001e3
[    0.000000] : level2_kernel_pgt[9] = 12001e3
[    0.000000] : level2_kernel_pgt[10] = 14001e3
[    0.000000] : level2_kernel_pgt[11] = 16001e3
[    0.000000] : level2_kernel_pgt[12] = 18001e3
[    0.000000] : level2_kernel_pgt[13] = 1a001e3
[    0.000000] : level2_kernel_pgt[14] = 1c001e3
[    0.000000] : level2_kernel_pgt[15] = 1e001e3
[    0.000000] : level2_kernel_pgt[16] = 20001e3
[    0.000000] : level2_kernel_pgt[17] = 22001e3
```

在cleanup_highmap()之前，level2_kernel_pgt中一共有256项有效项。你看是不是和之前计算的能对应上了。而且每个映射是线性的。

经过cleanup_highmap()之后，level2_kernel_pgt中只有[8..17]项是有效映射了。然后再来仔细分析一下。[8] = 0x1000000 这个值是不是正好是_pa(_text)？而[17] = 0x2200000 又和_brk_end对应上？

# 长这样

用图来看，应该能够更清楚一些

![这里写图片描述](/kernel_pagetalbe/pagetable_after_cleanup_highmap.png)

感觉离真相又近了一步～

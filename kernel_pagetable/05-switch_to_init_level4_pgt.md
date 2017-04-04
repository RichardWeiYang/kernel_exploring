内核启动阶段，使用的页表是early_level4_pgt。（有可能不是，大部分情况下是的。）

early_level4_pgt的设置可以看[head_64.S设置的段页][1]

看这个名字就能猜出来，这个页表仅仅是初始的时候使用的。那什么时候内核第一次更改的呢？

# 成功上位

```
void __init init_mem_mapping(void)
{

	load_cr3(swapper_pg_dir);
	__flush_tlb_all();

}
```

然后呢？ 咱来做个实验吧～

给出补丁～ 4.7上的

```
diff --git a/arch/x86/include/asm/pgtable_64.h b/arch/x86/include/asm/pgtable_64.h
index 2ee7811..f6dc1ed 100644
--- a/arch/x86/include/asm/pgtable_64.h
+++ b/arch/x86/include/asm/pgtable_64.h
@@ -21,6 +21,7 @@ extern pmd_t level2_fixmap_pgt[512];
 extern pmd_t level2_ident_pgt[512];
 extern pte_t level1_fixmap_pgt[512];
 extern pgd_t init_level4_pgt[];
+extern pgd_t early_level4_pgt[];

 #define swapper_pg_dir init_level4_pgt

diff --git a/arch/x86/mm/init.c b/arch/x86/mm/init.c
index 372aad2..ba19480 100644
--- a/arch/x86/mm/init.c
+++ b/arch/x86/mm/init.c
@@ -577,7 +577,7 @@ static void __init memory_map_bottom_up(unsigned long map_start,

 void __init init_mem_mapping(void)
 {
-	unsigned long end;
+	unsigned long end, pgd;

 	probe_page_size_mask();

@@ -619,7 +619,13 @@ void __init init_mem_mapping(void)
 	early_ioremap_page_table_range_init();
 #endif

+	printk(KERN_ERR "%s: early_level4_pgt is %lx\n", __func__, __pa_symbol(early_level4_pgt ));
+	printk(KERN_ERR "%s: init_level4_pgt  is %lx\n", __func__, __pa_symbol(init_level4_pgt ));
+	pgd = read_cr3();
+	printk(KERN_ERR "%s: current cr3 is %lx\n", __func__, pgd);
 	load_cr3(swapper_pg_dir);
+	pgd = read_cr3();
+	printk(KERN_ERR "%s: cr3 is set  to %lx\n", __func__, pgd);
 	__flush_tlb_all();

 	early_memtest(0, max_pfn_mapped << PAGE_SHIFT);
```

其实很简单，就是打印了一下early_level4_pgt, init_level4_pgt，用来对比一下cr3中，原有的值和更改后的值。

简单吧，然后看一下这次输出的结果，看看我们的猜想对不对～

```
[    0.000000] init_mem_mapping: early_level4_pgt is 1fca000
[    0.000000] init_mem_mapping: init_level4_pgt  is 1e06000

[    0.000000] init_mem_mapping: current cr3 is 1fca000
[    0.000000] init_mem_mapping: cr3 is set  to 1e06000
```

好了，内核页表切换完成。进入到init_level4_pgt统治的时代了。

# 打印页表

页表切换了，那都切换成什么样子了呢？

先来看一下代码

```
diff --git a/arch/x86/mm/init.c b/arch/x86/mm/init.c
index 372aad2..49acd9a 100644
--- a/arch/x86/mm/init.c
+++ b/arch/x86/mm/init.c
@@ -577,7 +577,8 @@ static void __init memory_map_bottom_up(unsigned long map_start,

 void __init init_mem_mapping(void)
 {
-	unsigned long end;
+	unsigned long end, pud;
+	int i, j;

 	probe_page_size_mask();

@@ -620,6 +621,18 @@ void __init init_mem_mapping(void)
 #endif

 	load_cr3(swapper_pg_dir);
+	for (i = 0; i < 512; i++) {
+		if (init_level4_pgt[i].pgd) {
+			printk(KERN_ERR "%s: init_level4_pgt[%d] 0x%lx\n", __func__, i, init_level4_pgt[i].pgd);
+			pud = init_level4_pgt[i].pgd;
+			pud = (unsigned long)__va(pud & 0xfffff000);
+			for (j = 0; j < 512; j++) {
+				if (*(unsigned long *)(pud + j * 8)) {
+					printk(KERN_ERR "%s: \t pud[%d] = %lx\n", __func__, j, *(unsigned long *)(pud + j*8));
+				}
+			}
+		}
+	}
 	__flush_tlb_all();

 	early_memtest(0, max_pfn_mapped << PAGE_SHIFT);
```

其实就是遍历init_level4_pgt，如果不是空就再往下走一层。

那来看看打出来的是啥

```
[    0.000000] init_mem_mapping: init_level4_pgt[272] 0x2047067
[    0.000000] init_mem_mapping: 	 pud[0] = 2048067
[    0.000000] init_mem_mapping: 	 pud[1] = 13ffff067
[    0.000000] init_mem_mapping: 	 pud[2] = 13fffe067
[    0.000000] init_mem_mapping: 	 pud[4] = 204a067
[    0.000000] init_mem_mapping: init_level4_pgt[511] 0x1c09067
[    0.000000] init_mem_mapping: 	 pud[510] = 1c0a063
[    0.000000] init_mem_mapping: 	 pud[511] = 1c0b067
```

恩，先看后面这init_level4_pgt[511]的内容。回顾一下这张图。

![这里写图片描述](/kernel_pagetable/switch_to_init_level4_pgt.png)

再来看这段代码。

```
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data)
{
	...

	clear_page(init_level4_pgt);

	...

	/* set init_level4_pgt kernel high mapping*/
	init_level4_pgt[511] = early_level4_pgt[511];

	...
}
```

在这段代码中，init_level4_pgt[511]设置成了early_level4_pgt一样的值。
而在early_level4_pgt[511]保存的是level3_kernel_pgt，继而又保存了level2_kernel_gpt。而他们俩正好差一个页。


[1]: http://blog.csdn.net/richardysteven/article/details/52629731#t16

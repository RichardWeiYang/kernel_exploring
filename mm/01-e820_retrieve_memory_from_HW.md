内存，从硬件上看到的是一根根有着金手指的内存条，从软件的角度上看就是编了号的地址。虽然从理论上讲内存的编号可以是任意的，但是和我们见到的门牌号码一样，地址是有范围的，甚至中间可能是有空洞的。

在我们真正能访问内存之前，先得有人获取真实的物理地址。就好像人口普查时，专门有人挨家挨户登记入住情况。

在x86平台上，这个工作就交给了e820。

# 神马是e820

[Definition from Wikipedia:] [1]

> e820 is shorthand to refer to the facility by which the BIOS of x86-based computer systems reports the memory map to the operating system or boot loader.
>
> It is accessed via the int 15h call, by setting the AX register to value E820 in hexadecimal. It reports which memory address ranges are usable and which are reserved for use by the BIOS.

偶来总结一下下：

* e820 是一个可以侦测物理内存分布的硬件
* 软件通过15号中断与之通信

这就意味着在x86平台上，如果想要知道内存分布，那就要和e820沟通。据我所知，不同的平台使用不同的方式，比如ppc64平台上使用的是device tree。

不管怎么样，系统若要正常运行，需要通过某种方式获得正确的内存分布，这才是需要关注的重点。

## e820表

从软件的角度看，e820就一张保存了内存布局的表。如果使用的是linux系统，可以输入下列的命令观察：

```bash
dmesg | grep e820

```

输出结果如下：

```
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000feffc000-0x00000000feffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

你看，是不是挺简单的。其实就是一个表。每一个表项表示了一个内存空间，包含了起始地址，结束地址和类型。下面就是这个表项的数据结构。是不是觉得非常简单～

```c
struct e820entry {
	__u64 addr;	/* start of memory segment */
	__u64 size;	/* size of memory segment */
	__u32 type;	/* type of memory segment */
} __attribute__((packed));
```


# 如何使用e820

知道了e820的作用，那现在来看看内核是怎么使用它的。

其实说白了很简单，先从硬件读取数据保存成e820表，再经过一定的排序合并保存在内核中。当然这只是内存信息的原始数据。就好像人口普查得到的最初的一手数据。该信息本事是为了下一步的处理做准备的。

## 从硬件获得e820表

当然了，最关键的就是从硬件读取e820表的结构，这个函数就是detect_memory_e820().

```c
static int detect_memory_e820(void)
{
	int count = 0;
	struct biosregs ireg, oreg;
	struct e820entry *desc = boot_params.e820_map;
	static struct e820entry buf; /* static so it is zeroed */

	initregs(&ireg);
	ireg.ax  = 0xe820;
	ireg.cx  = sizeof buf;
	ireg.edx = SMAP;
	ireg.di  = (size_t)&buf;

	/*
	 * Note: at least one BIOS is known which assumes that the
	 * buffer pointed to by one e820 call is the same one as
	 * the previous call, and only changes modified fields.  Therefore,
	 * we use a temporary buffer and copy the results entry by entry.
	 *
	 * This routine deliberately does not try to account for
	 * ACPI 3+ extended attributes.  This is because there are
	 * BIOSes in the field which report zero for the valid bit for
	 * all ranges, and we don't currently make any use of the
	 * other attribute bits.  Revisit this if we see the extended
	 * attribute bits deployed in a meaningful way in the future.
	 */

	do {
		intcall(0x15, &ireg, &oreg);
		ireg.ebx = oreg.ebx; /* for next iteration... */

		/* BIOSes which terminate the chain with CF = 1 as opposed
		   to %ebx = 0 don't always report the SMAP signature on
		   the final, failing, probe. */
		if (oreg.eflags & X86_EFLAGS_CF)
			break;

		/* Some BIOSes stop returning SMAP in the middle of
		   the search loop.  We don't know exactly how the BIOS
		   screwed up the map at that point, we might have a
		   partial map, the full map, or complete garbage, so
		   just return failure. */
		if (oreg.eax != SMAP) {
			count = 0;
			break;
		}

		*desc++ = buf;
		count++;
	} while (ireg.ebx && count < ARRAY_SIZE(boot_params.e820_map));

	return boot_params.e820_entries = count;
}
```

略长，不过逻辑很简单。内核通过15号中断遍历整个表，并把它保存到了boot_params.e820_map。


## 保存表项到内核区域

从刚才的代码中看到，取出的表项放在boot_params中，并不安全，而且取出的表项也没有排序。所以接下来，内核把数据拷贝到指定的数据结构，并排序整个表。

这个步骤在函数default_machine_specific_memory_setup()中完成。

```c
char *__init default_machine_specific_memory_setup(void)
{
	char *who = "BIOS-e820";
	u32 new_nr;
	/*
	 * Try to copy the BIOS-supplied E820-map.
	 *
	 * Otherwise fake a memory map; one section from 0k->640k,
	 * the next section from 1mb->appropriate_mem_k
	 */
	new_nr = boot_params.e820_entries;
	sanitize_e820_map(boot_params.e820_map,
			ARRAY_SIZE(boot_params.e820_map),
			&new_nr);
	boot_params.e820_entries = new_nr;
	if (append_e820_map(boot_params.e820_map, boot_params.e820_entries)
	  < 0) {
		u64 mem_size;

		/* compare results from other methods and take the greater */
		if (boot_params.alt_mem_k
		    < boot_params.screen_info.ext_mem_k) {
			mem_size = boot_params.screen_info.ext_mem_k;
			who = "BIOS-88";
		} else {
			mem_size = boot_params.alt_mem_k;
			who = "BIOS-e801";
		}

		e820.nr_map = 0;
		e820_add_region(0, LOWMEMSIZE(), E820_RAM);
		e820_add_region(HIGH_MEMORY, mem_size << 10, E820_RAM);
	}

	/* In case someone cares... */
	return who;
}
```

关键的细节不在这个函数中，而是在:
* sanitize_e820_map(), 对整个表排序
* append_e820_map(), 把整个表拷贝到指定的数据结构

# 启动顺序

现在我们来看一下内核在什么时候，什么顺序执行以上两个步骤。这样你会有一个全局观。

```
    /* Retrieve the e820 table from BIOS */
    main(), arch/x86/boot/main.c, called from arch/x86/boot/header.S
          detect_memory()
               detect_memory_e820(), save bios info to boot_params.e820_entries

    /* store and sort e820 table to kernel area */
    start_kernel()
          setup_arch()
               setup_memory_map(),
                     default_machine_specific_memory_setup(), store and sort e820 table
                     e820_print_map()
```

最后的这个e820_print_map()函数就是打印出我们使用 "dmesg | grep e820"抓取到的信息的函数了。

Well, that's it~

# 重要的APIs

接下来我们看一下几个重要的API，内核就是使用这些API来操作e820表的。 所有这些函数都定义在"arch/x86/kernel/e820.c".

如果只想对e820有个初步认识，那到这里就可以结束了。我们已经看到了表的样子和获取排序表的基本流程。接下去是更深入的细节，对实现有兴趣的话建议在电脑上阅读。

## e820_add_region()/e820_remove_range()

他们正好是一对操作e820表的函数，本别用来添加和删除一段区域。重要的是，他们并不保证表项之间的顺序关系。

## __e820_update_range()

更新e820表项中的"type"属性。

比如，从 E820_RAM 变到 E820_RESERVED.

## e820_search_gap()

这个函数用来寻找内存区域之间的空洞，在x86平台上用来寻找设备的 MMIO空间。

## memblock_x86_fill()

这个函数将e820的数据结构转换成memblock的数据结构。这个动作非常重要，因为memblock是在启动初期没有页分配器时使用的内存分配模块。

我们将在另一个主题上讨论memblock。

# 一些有趣的算法

## sanitize_e820_map()

这是用来排序从硬件中获取的e820表项的函数。

### 每个e820entry由两个change_member表述

    e820entry                         change_member
    +-------------------+             +------------------------+
    |addr, size         |             |start, e820entry        |
    |                   |     ===>    +------------------------+
    |type               |             |end,   e820entry        |
    +-------------------+             +------------------------+

### 整个表形成了一个change_member的数组

     change_member*[]                               change_member[]
     +------------------------+                     +------------------------+
     |*start1                 |      ------>        |start1, entry1          |
     +------------------------+                     +------------------------+
     |*end1                   |      ------>        |end1,   entry1          |
     +------------------------+                     +------------------------+
     |*start2                 |      ------>        |start2, entry2          |
     +------------------------+                     +------------------------+
     |*end2                   |      ------>        |end2,   entry2          |
     +------------------------+                     +------------------------+
     |*start3                 |      ------>        |start3, entry3          |
     +------------------------+                     +------------------------+
     |*end3                   |      ------>        |end3,   entry3          |
     +------------------------+                     +------------------------+
     |*start4                 |      ------>        |start4, entry4          |
     +------------------------+                     +------------------------+
     |*end4                   |      ------>        |end4,   entry4          |
     +------------------------+                     +------------------------+

### 对change_member->addr排序

     change_member*[]                
     +------------------------+      
     |*start1                 |      
     +------------------------+      
     |*start2                 |      
     +------------------------+      
     |*end2                   |      
     +------------------------+      
     |*end1                   |      
     +------------------------+      
     |*start3                 |      
     +------------------------+      
     |*start4                 |      
     +------------------------+      
     |*end4                   |      
     +------------------------+      
     |*end3                   |      
     +------------------------+      

### 遍历排序后的change_member* 数组

因为chage_mamber已经排序了，所以遍历数组就得到了整个内存空间从小到大的顺序。

比如我们现在排序后是这样的结果。

```
      (start1,   start2,   end2,   end1,   start3,   start4,   end4,   end3)
```

#### 用overlap_list跟踪 "type"的变化

overlap_list保存了遍历过程中没有配对的change_member。每次有变动都会计算一下整个数组中的type属性。如果有变化，则表明有内存区域的临界值出现。

下面是手动展开的一个例子:

```
* chgidx == 0

     change_member*[]                      overlap_list[]
     +------------------------+            +-------------------+
 --> |*start1                 |            |*start1            |
     +------------------------+            +-------------------+
     |*start2                 |      
     +------------------------+      
     |*end2                   |      
     +------------------------+      
     |*end1                   |      
     +------------------------+      
     |*start3                 |      
     +------------------------+      
     |*start4                 |      
     +------------------------+      
     |*end4                   |      
     +------------------------+      
     |*end3                   |      
     +------------------------+      


* chgidx == 1

     change_member*[]                      overlap_list[]
     +------------------------+            +-------------------+
     |*start1                 |            |*start1            |
     +------------------------+            +-------------------+
 --> |*start2                 |            |*start2            |
     +------------------------+            +-------------------+
     |*end2                   |      
     +------------------------+      
     |*end1                   |      
     +------------------------+      
     |*start3                 |      
     +------------------------+      
     |*start4                 |      
     +------------------------+      
     |*end4                   |      
     +------------------------+      
     |*end3                   |      
     +------------------------+      

* chgidx == 2

     change_member*[]                      overlap_list[]
     +------------------------+            +-------------------+
     |*start1                 |            |*start1            |
     +------------------------+            +-------------------+
     |*start2                 |            |                   |
     +------------------------+            +-------------------+
 --> |*end2                   |      
     +------------------------+      
     |*end1                   |      
     +------------------------+      
     |*start3                 |      
     +------------------------+      
     |*start4                 |      
     +------------------------+      
     |*end4                   |      
     +------------------------+      
     |*end3                   |      
     +------------------------+      
```

有没有对 overlap_list[] 的工作原理清楚一些？

overlap_list[]的变化一共有两种情况
* 遇见 "start", 则加入overlap_list[].
* 遇见 "end", 则将对应的"start"从overlap_list[]中删除

"current_type" 整个overlap_list[]数组中type属性的最大值. 而 "last_type" 上一个循环中的"current_type"。 当这两个值不同时，表明了我们遇到了e820数组的边界，这时候就需要添加一个或者设置适当的大小了。

让我再用下面这张图来解释一下这个过程。

```
0     type1      type2     type1   0       type3     type4     type4   type3
        |          |         |      |        |         |        |       |
        v          v         v      v        v         v        v       v
      (start1,   start2,   end2,   end1,   start3,   start4,   end4,   end3)
```

## e820_search_gap()

这个函数用来搜索e820中最大的空洞。在x86平台上用来查找设备的MMIO空间。

逻辑比较简单

```
i = nr_map - 1;
                                                             end           last
                                                              |<--- gap  --->|
      (start1, end2)         (start2, end2)         (start3, end3)


i = nr_map - 2;
                                      end           last
                                       |<--- gap  --->|
      (start1, end2)         (start2, end2)         (start3, end3)

i = nr_map - 3;
                end           last
                 |<--- gap  --->|
      (start1, end2)         (start2, end2)         (start3, end3)
```

每次迭代计算出两个e820表项之间的空洞，最后返回满足要求的最大空洞。

顺便提一下，到v4.9内核，该函数有一个bug。也就是并不能准确找到满足要求的最大空洞。我的修改意见如下，不过Yinghai觉得这个函数没有别人用了，所以不需要这么去算了。


```
From: Wei Yang <richard.weiyang@gmail.com>
Date: Sun, 18 Dec 2016 22:16:52 +0800
Subject: [PATCH] x86/e820: fix e820_search_gap() in case the start_addr is a
 gap

For example, we have a e820 map like this:

    e820: [mem 000000000000000000-0x000000000000000f] usable
    e820: [mem 0x00000000e0000000-0x00000000e000000f] usable

The biggest gap should be:

    [mem 0x00000010-0xdfffffff]

While if start_addr is passed by a value bigger than 0x10, current version
will tell following range is the biggest gap:

    [mem 0xe0000010-0xffffffff]

The patch fixes this by set end to start_addr, when start_addr is bigger
than end.

Signed-off-by: Wei Yang <richard.weiyang@gmail.com>
---
 arch/x86/kernel/e820.c | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

diff --git a/arch/x86/kernel/e820.c b/arch/x86/kernel/e820.c
index f4fb197..1826591 100644
--- a/arch/x86/kernel/e820.c
+++ b/arch/x86/kernel/e820.c
@@ -602,7 +602,7 @@ __init int e820_search_gap(unsigned long *gapstart, unsigned long *gapsize,
 		unsigned long long end = start + e820->map[i].size;

 		if (end < start_addr)
-			continue;
+			end = start_addr;

 		/*
 		 * Since "last" is at most 4GB, we know we'll
--
2.5.0
```

所以最后的补丁长这个样子 [make e820_search_gap() static][2].

[1]: https://en.wikipedia.org/wiki/E820
[2]: https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=b4ed1d15b453c86b4b9362128bd7a0ecd95a105c

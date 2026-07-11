在内核中这个管理的核心单位是 **页(page)**，所以对内存的管理可以从大方向上分成两个步骤：

  * 准确的划分出页
  * 高效得使用好页

在前几章的内容中，我们已经一步步完成了第一个步骤的任务。尤其是在[寻找页结构体的位置][1]一章我们已经看到了当前常用的sparsemem是如何关联一个物理地址(pfn)和一个页面(page struct)的。

然而我们如果仔细观察，在整个内存管理中除了页，为了高效管理内存还有不少其他粒度的管理单位。这次我们从粒度的角度来探究一下内存管理的层次，也顺便补充一下几个鲜为人知的内存管理单元。

按照层次，不考虑node和zone的概念，整个系统上的内存从下到上按照粒度划分可以有这么几个层次。

```
              +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 slub         | | | | | | | | | | | | | | | | | | | | | | | | | | | | | | | | |
              +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                                            
 page(buddy)  |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |                                            
              +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 page_block   |       |       |       |       |       |       |       |       |
              +===+===+===+===+===+===+===+===+===+===+===+===+===+===+===+===+
 subsection   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
              +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 mem_section  |               |               |               |               |
              +---------------+---------------+---------------+---------------+
 memory_block |                               |                               |
              +-------------------------------+-------------------------------+
 memblock     |                                                               |
              +---------------------------------------------------------------+
 e820         |                                                               |
              +---------------------------------------------------------------+
```

从这张图中我们可以看到，为了管理好内存，内核开发者将整个内存分成了不同的层次不同的粒度来管理。

究竟这几个粒度在系统中起到什么作用，以及之间的关系,让我们来一探究竟。

另外值的注意的是，有些层次有管理单元的最小大小, 因为大小的重叠可能会有些潜在问题。这里记录一下大小，便于后续查找：

  * page: 最大有MAX_PAGE_ORDER, 必须 >= PAGE_BLOCK_MAX_ORDER
  * page_block: 根据配置不同而不同，比如是HPAGE_PMD_ORDER和PAGE_BLOCK_MAX_ORDER的最小值。默认是2^10个页大小，算4M
  * subsection: 2M, (SUBSECTION_SHIFT 21)
  * mem_section:  x86上是128M, (SECTION_SIZE_BITS	27)
  * memory_block: 最小是一个mem_section, (MIN_MEMORY_BLOCK_SIZE)

# 粒度单位的分水岭

在上面几个粒度层次划分中，有一个非常重要的分水岭 -- **page_block**。

  * 往上：划分单位是页的数目，通常是 2的多少次方页
  * 往下：划分单位是原始内存大小，比如MB

所以这里有个麻烦的地方，比如subsection定义的是2M，默认配置情况，也就是4K页大小，page_block大小是4M。

但是如果页大小配置为64K，那page_block大小就是64M。也就会夸更多个subsection，这样对某些情况下的校验，提出了更高的要求。

# e820

e820是x86影响平台上报内存布局的硬件，对软件处理来说，我们几乎可以忽略这部分。

这部分在[e820][2]有详细介绍。

# memblock

memblock是第一层软件对硬件上报内存布局的抽象，大部分情况我们在系统运行起来后，也不太接触到这个。

这部分在[memblock][3]有详细介绍。

# memory_block

这个东西呢说有用也有用，说没用呢也没用。因为大多数人平时也用不到。就我现在所知，它啊在内存热插拔的时候才会用到。但是呢他们代表了系统上所有的内存设备。

在系统上我们其实能看到这个东东，因为他们都在 /sys/devices/system/memory/ 这个目录下。

## 初始化流程

对于系统上所有的内存，我们都有对应的memory_block设备。那这些设备是怎么生成的呢？我们来看一下初始化的代码吧。

```
    memory_dev_init(), called kernel_init_freeable->do_basic_setup->driver_init()
        block_sz = memory_block_size_bytes()
        sections_per_block = block_sz / MIN_MEMORY_BLOCK_SIZE
        subsys_system_register(&memory_subsys, memory_root_attr_groups)
        add_memory_block(nr), nr equals the start section number of memory_block
            init_memory_block(mem, base_memory_block_id(nr), MEM_ONLINE)
                mem->start_section_nr = block_id * sections_per_block
                mem->state = MEM_ONLINE
                mem->nid = NUMA_NO_NODE
                register_memory(mem), mem->dev.bus = &memory_subsys
```

怎么样，其实很简单吧。就是每block_sz大小就会对应有一个memory_block的设备。但是这个block_sz的大小不是固定的，**最小的大小是一个section**。

嗯，貌似讲完了，但是实际上还没有。上面这段代码是在初始化时做的操作，但是在内存热插拔的时候还会有不同的操作。

## 热插拔何时创建设备

既然对所有的内存都会有这个一个设备，那么热插拔的时候也会生成这么一个设备。所以我们来看热插时的这个流程。

```
      add_memory()
          add_memory_resource()
              create_memory_block_devices()
                  init_memory_block()
```

是不是也挺简单的。别急，我再告诉你个东东 -- online。

内存热插进来后其实还不能被内核使用，准确的说是不能被buddy使用。此时还差一步，那就是online了。
（嗯。。。这个名字其实不是很好理解。）

那这个过程是如何触发的呢？ 有两种情况

  * 默认直接上线
  * 手动上线

默认直接上线的我们就不看了，来看这个手动上线的。 还记得刚才我们是在系统的那个目录下看到memory_block设备的么？

对了，在 /sys/devices/system/memory/ 下。然后你找一个具体的设备，再进去就能看到一个文件**online**.

这个文件对应的内核操作是什么呢？还记得注册的subsys的变量么？对了就是 memory_subsys。

```
static struct bus_type memory_subsys = {
	.name = MEMORY_CLASS_NAME,
	.dev_name = MEMORY_CLASS_NAME,
	.online = memory_subsys_online,
	.offline = memory_subsys_offline,
};
```

我把它老人家放在这里了，我想你能才到上线时触发到了谁吧～

# mem_section

mem_section在[寻找页结构体][4]中做了比较详细的描述。

值得注意的是，subsection/pageblock的定义也在mem_section中。都位于mem_section.usage中。

  * subsection_map
  * pageblock_flags

而这两者在管理粒度上有着重要的差异：

```
    subsection            |          pageblock
    ===================================================
    静态                  | 动态
    ----------------------+----------------------------
    划分单位为MB          | 划分单位为page_number
    ---------------------------------------------------
```

## subsection

subsection在sparse_init()中初始化完成，并通过sparse_init_subsection_map()标记了存在的内存后，就不会再变化了。

有一点值得注意的是，subsection_mask_set()过程中，是允许有空缺的。

## pageblock

但是pageblock则会在运行中根据状态变化，并且其状态是跟随者再上层的page变化的。所以它的划分单位是页的个数。这样导致了pageblock所包含的subsection个数在不同配置中不是一个固定值。

通常情况下，pageblock的大小是2^10个页面大小，也就是4M。可以包含两个subsection。

这部分在[pageblock和migrationtype][5]中有详细描述。

[1]: /mm/03-sparsemem.md
[2]: /mm/01-e820_retrieve_memory_from_HW.md
[3]: /mm/02-memblock.md
[4]: /mm/03-sparsemem.md
[5]: /mm/15-pageblock_migratetype.md

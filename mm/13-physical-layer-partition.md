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
 page         |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |                                            
              +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 page_block   |       |       |       |       |       |       |       |       |
              +-------+-------+-------+-------+-------+-------+-------+-------+
 mem_section  |               |               |               |               |
              +---------------+---------------+---------------+---------------+
 memory_block |                               |                               |
              +-------------------------------+-------------------------------+
 memblock     |                                                               |
              +---------------------------------------------------------------+
 e820         |                                                               |
              +---------------------------------------------------------------+
```

从这张图中我们可以看到，为了管理好内存，内核开发者将整个内存分成了不同的层次不同的粒度来管理。其中page和slub已经有很多文章来解读，而e820, memblock和mem_section在前面的章节中已有描述，本章就不在赘述。这里我们着重看看通常被忽略的两个粒度：

  * memory_block
  * pageblock

究竟这两个粒度在系统中起到什么作用呢？让我们来一探究竟。

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

# pageblock

对pageblock的使用小编暂时还不是特别理解，主要的用途之一和内存迁移相关。这里我们先从原理上理解pageblock的作用，至于用途等到小编后续学习了之后再来讲解。

## pageblock的位置

pageblock这个概念其实有点尴尬，因为它不像其他的内存粒度有一个非常明确的结构体来表达。对pageblock的描述只是一个挂在mem_section结构体上的位图(bitmap)。

多说无益，来张图看看：

```
     mem_section
     +-----------------------------+
     |section_mem_map              |
     |   (unsigned long)           |
     |                             |
     +-----------------------------+
     |usage                        |
     |   (mem_section_usage *)     |
     |   +-------------------------+
     |   |subsection_map           |
     |   |    (bitmap)             |
     |   |                         |
     |   |                         |
     |   |                         |
     |   |pageblock_flags[0]       |    [(1UL << (PFN_SECTION_SHIFT - pageblock_order))]
     |   |    (unsigned long)   ---|--->+----+----+----+---...---+----+----+
     +---+-------------------------+    |    |    |    |         |    |    |
                                        |    |    |    |         |    |    |
                                        +----+----+----+---...---+----+----+
```

从上图中可以看到，在每个mem_section结构体上用一个pageblock_flags的位图，来表示对应物理内存的属性。

为了节省空间（小编猜的），并不是给每一个页分配对应的属性，而是每pageblock_order个页共用一个pageblock属性。这个pageblock_order有几种不同的可能性，其中一种就是buddy分配器上的最大页面值。

```
#define pageblock_order		(MAX_ORDER-1)
```

从这个角度看，pageblock的粒度就是 MAX_ORDER个页。

## pageblock的定义

上面我们看到了pageblock是一个位图，那这个位图上究竟定义的是什么，每一位代表什么意义？

待小编画个草图给看官瞧一瞧：

```
    3     2     1     0      total NR_PAGEBLOCK_BITS = 4
   +-----+-----+-----+-----+
   |     |     |     |     |
   |     |     |     |     |
   +-----+-----+-----+-----+
      |     |     |     |
      |     |     |     +---
      |     |     |          \
      |     |     +---------   = enum migratetype
      |     |                /
      |     +---------------
      |
      +---------------------  migrate_skip
```

也就是每个pageblock_order页面都有4bit的位图来表示其属性。其中最高位有点特殊，表示一个特殊含义。而后三位则是一个枚举类型migratetype。其可能的值定义为：

```
    enum migratetype {
    	MIGRATE_UNMOVABLE,
    	MIGRATE_MOVABLE,
    	MIGRATE_RECLAIMABLE,
    	MIGRATE_PCPTYPES,	/* the number of types on the pcp lists */
    	MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
    #ifdef CONFIG_CMA
    	MIGRATE_CMA,
    #endif
    #ifdef CONFIG_MEMORY_ISOLATION
    	MIGRATE_ISOLATE,	/* can't allocate from here */
    #endif
    	MIGRATE_TYPES
    };
```

嗯，是啥意思咱先不管了，至少看出来是怎么定义的。

在继续阅读代码之前，小编先出道题考一考大家。代码中有这么一段定义，大家能理解为什么要这么写么？

```
BUILD_BUG_ON(MIGRATE_TYPES > (1 << PB_migratetype_bits));
```

## pageblock的使用

说了这么一堆，定义了这么个位图，那究竟是怎么使用呢？嗯，其中的奥妙就要联系到buddy分配器了。在这里小编只讲解与pageblock相关的部分，其他的细节还请读者们自行网上搜索相关资料～

这其中的奥秘啊还要从free_area说起，小编将free_area中的相关部分摘取出来。

```
      +------------------------------+
      |free_area[MAX_ORDER]  0...11  |
      |   (struct free_area)         |
      |   +--------------------------+
      |   |free_list[MIGRATE_TYPES]  |
      |   |(struct list_head)        |
      +---+--------------------------+
```

所以我们之前看到的free_area->free_list其实不止一个链表，而是每个migrate_type一个链表。当我们去分配内存时，不仅要看大小上的匹配，还要看migrate_type的匹配。

当然，为了减少内存碎片，当我们找不到内存时，还可以在不同的migrate_type上去搜索。只是这个搜索还是有一定规则的，也不是阿猫阿狗都能用不同类型的空闲内存。

小编在这里只给出一个方向，具体的代码就留给各位想要进一步学习的读者自行探索啦～

  * find_suitable_fallback()
  * fallbacks[MIGRATE_TYPES]

[1]: /mm/03-sparsemem.md

ZONE_DEVICE牵扯到一个概念，[HMM(Heterogeneous Memory Management)][1]。

目前内核已经把这部分包装得比较好了，可以从两个角度去理解：

  * 初始化
  * 使用

# 初始化

和普通内存在系统中使用一样，设备内存也有对应的page struct。所以初始化就是做这部分工作，并且将对应的内存划分到ZONE_DEVICE。

PS: ZONE_DEVICE内存比较特殊，在系统启动的过程中并没有这部分内存，而是通过对应的驱动，借助了热插拔的逻辑添加进系统的。这点很有意思，因为这部分不会影响到启动内存在Zone上的划分。

主要的接口是

  * devm_memremap_pages()
  * memremap_pages(struct dev_pagemap*, int nid)

前者比后者多了一个devm_add_action_or_reset()，注册了一个设备释放时清理对应内存的函数 -- devm_memremap_pages_release。所以总得来说，初始化的核心还是memremap_pages()。

其中struct dev_pagemap包含了和设备内存相关的重要信息。

## struct dev_pagemap

```
   struct dev_pagemap
   +---------------------+
   |type                 |  内存类型，如：device private
   |   (enum memory_type)|
   |owner                |  指向拥有设备内存的真实设备
   |   (void *)          |
   |altmap               |
   |  (strct vmem_altmap)|
   |   +-----------------+
   |   |base_pfn         |
   |   |end_pfn          |
   |   |    (unsigned l) |
   |   |                 |
   |   +-----------------+
   |ops                  |
   |   (dev_pagemap_ops*)|
   |   +-----------------+
   |   |folio_free       |  设备内存释放
   |   |migrate_to_ram   |  将设备内村复制到系统内存，fault时调用
   |   |                 |
   |   +-----------------+
   |nr_range             |
   |   (int )            |
   |range[]              |
   |   (struct range)    |
   |   +-----------------+
   |   |start            |  物理地址
   |   |end              |
   |   |    (u64)        |
   +---+-----------------+
```

解释一下其中重要的信息：

  * type - 设备内存的类型
  * nr_range/range - 设备内存包含的地址空间范围
  * ops - 内存迁移释放的回调函数

其中值的注意的是，设备内存还分为几种类型，

    * MEMORY_DEVICE_PRIVATE
    * MEMORY_DEVICE_COHERENT
    * MEMORY_DEVICE_FS_DAX
    * MEMORY_DEVICE_GENERIC
    * MEMORY_DEVICE_PCI_P2PDMA

其中MEMORY_DEVICE_PRIVATE这个类型，CPU不能直接访问，所以也没有建立内核页表的映射，但是有page struct。 而且也是通过move_pfn_range_to_zone()，“移动”到Zone Device并初始化每一个page。

## 流程

```
memremap_pages(pgmap, nid)
    struct mhp_params params = {
      .altmap = pgmap_altmap(pgmap),
      .pgmap = pgmap,
      .pgprot = PAGE_KERNEL,
    };

    for_each(pgmap->nr_range)
        pagemap_range(pgmap, &params, i, nid)
            range = pgmap->ranges[i]

            ... 一系列检查
            mem_hotplug_begin()

            add_pages(nid, range->start, range_len(range), params)             // 差别在与有没有内核页表映射
            ...or...
            arch_add_memory(nid, range->start, range_len(range), params)

            move_pfn_range_to_zone(ZONE_DEVICE, start, len, params->altmap, )  --- 第一次初始化

            mem_hotplug_done()

            memmap_init_zone_device(ZONE_DEVICE, start, len, pgmap)            --- 第二次初始化
```

但是有意思的是，pagemap_range()在两处初始化了page struct, (1)和(2)。所以为什么还会再用memmap_init_zone_device()再初始化一次？

其实不是的，而是分了两个阶段来初始化

    * memmap_init_range(): 初始化位于头部的必要部分
    * memmap_init_zone_device(): 释放掉hotplug锁后，初始化剩下的部分

但是这部分page struct并没有放到buddy allocator中,且设为为保留状态，__SetPageReserved。

```
   folio
   +--------------------------+
   |pgmap                     |
   |    (struct dev_pagemap *)|
   |    +---------------------+
   |    |type                 |  内存类型，如：device private
   |    |   (enum memory_type)|
   |    |owner                |  指向拥有设备内存的真实设备
   |    |   (void *)          |
   |    |ops                  |
   |    |   (dev_pagemap_ops) |
   |    |   +-----------------+
   |    |   |folio_free       |  设备内存释放
   |    |   |migrate_to_ram   |  将设备内村复制到系统内存，fault时调用
   |    |   |                 |
   |    +---+-----------------+
   |zone_device_data          |
   |    (void *)              |
   +--------------------------+
```

并且在对应的folio上设置了pgmap/zone_device_data。

其中pgmap中的ops->migrate_to_ram，用来将设备上内存内容搬移回系统内存中。

# 使用

这部分是单独为了 MEMORY_DEVICE_PRIVATE 类型出现的。没有理解错的话，其他类型的设备内存在初始化时建立了内核页表的映射，也就意味着内核可以直接访问设备内存。

但MEMORY_DEVICE_PRIVATE类型不行，所以需要单独处理。总体思想是将用户指定区域的系统内存“迁移”到设备内存中。

在进程的地址空间，会划分出一段区域作为系统和设备的“共享空间”。但是在同一时间，其中之一可以使用这块区域。

下面是这个过程中，某一地址页表变化的过程

```

                              +--------------------+
                              |Migration Entry     | with RAM PFN
                              +--------------------+
                            ^                        \
       migrate_vma_setup() /                          \ migrate_vma_finalize()
                          /                            v
   +--------------------+                               +--------------------+
   |System Ram          |                               |DevicePrivate Entry |
   +--------------------+                               +--------------------+
                          ^                            /
    migrate_vma_finalize() \                          / migrate_vma_setup()
                            \                        v
                              +--------------------+
                              |Migration Entry     | with Device Private Folio PFN
                              +--------------------+

```

也就是不管是把系统内存迁移到设备，还是反过来，都经过一样的流程。 其中分为三个阶段：

  * migrate_vma_setup(): 收集指定进程地址范围内的pfn,并将对应页表设置为migration entry
  * migrate_vma_pages(): 在此之前，已经分配好设备内存或系统内存，并将用户内存的元信息复制到设备内存上
  * migrate_vma_finalize(): 将migrate_vma_setup()中设置的migration entry替换成新的内存地址

有意思的是migrate_vma_finalize()。这就替换在migrate_vma_setup()中设置为migration entry的页表的过程。

  * 系统->设备: 设置成pmd_is_device_private_entry()
  * 设备->系统: 设置成普通的entry

另外还有一个帮手：

  * hmm_range_fault: 获取用户进程对应范围的pfn

驱动可以通过hmm_range_fault()遍历指定范围的页表来获得用户进程内所指向的页(pfn)。如果用户没有分配内存(还没有fault)，hmm_range_fault()也会调用handle_mm_fault()填充。

[1]: https://www.kernel.org/doc/html/latest/mm/hmm.html

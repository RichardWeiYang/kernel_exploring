# 使能

cma是一个可配置功能，可以检查CMA这个内核配置有没有打开。

另外还有一个CMA_AREAS的配置参数，默认是20。

# 数据结构

```
struct cma cma_areas[MAX_CMA_AREAS];

cma_init_reserved_mem(base, size, order_per_bit, name, res_cma)

    cma
    +---------------------------+
    |count                      |  number of pages
    |available_count            |  = size >> PAGE_SHIFT
    |order_per_bit              |  order_per_bit
    |                           |
    |nr_ranges                  |  = 1
    |ranges[CMA_MAX_RANGES]     |
    |    (struct cma_memrange)  |
    |    +----------------------+
    |    |base_pfn              |  = PFN_DOWN(base)
    |    |early_pfn             |  = PFN_DOWN(base)
    |    |count                 |  = cma->count
    |    |                      |
    |    |bitmap                |
    |    +----------------------+
    |    |                      |
    |    |                      |
    +----+----------------------+
```

# 初始化及过程

cma是个有意思的东西。他先要通过memblock将需要的内存区域reserve，这样让别人用不到。然后再把这部分内存释放到buddy系统，好让系统使用。

在使用方需要的时候，再把这部分内存，如果已经被人用的情况下，迁移到空闲内存后，再使用。

## 创建预留

创建预留可以分成三步

  * 比如在__cma_declare_contiguous_nid()中先通过memblock预留下需要的空间
  * 再通过cma_init_reserved_mem()分到一个cma区域，分到时如上图中所示。
  * 然后在cma_init_reserved_areas()初始化，并释放到buddy

其中cma_init_reserved_areas()会调用init_cma_reserved_pageblock()将page释放到buddy。并且标注为MIGRATE_CMA类型。

## 分配

cma_alloc()

## 释放

cma_release()

# 使用的例子

目前默认使用cma的用户是dma，在内核启动过程中会调用dma_contiguous_reserve()分配连续内存。

这里值的注意的是调用的时机，在x86上，我们可以看到调用的时机在刚确认memblock numa后，在分配memmap前。

具体可以参考[memblock][1]中的全局流程。


[1]: /mm/02-memblock.md

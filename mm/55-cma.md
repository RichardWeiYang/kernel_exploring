# 全局流程

```
start_kernel()
    setup_arch()
        initmem_init() -> x86_numa_init() -> numa_init()

        dma_contiguous_reserve(max_pfn_mapped << PAGE_SHIFT)  ...  1 预留

        x86_init.paging.pagetable_init() -> paging_init()
            sparse_init()
                memblock_presents()
                    memory_present()                 // 分配mem_section，并标记present
                sparse_init_nid()
                sparse_init_subsection_map()         // 初始化subsection_map
            zone_size_init()
                free_area_init()                     // 初始化pgdat
    rest_init()
        kernel_init()
            kernel_init_freeable()
                smp_init()                           // 在这之前只有单线程, memblock分配只能发生在这之前
                page_alloc_init_late()
                do_basic_setup()
                    do_initcalls()
                        cma_init_reserved_areas()             ...  2 释放
```

完整的内存初始化流程可以参考[memblock][1]

cma完整的使用流程可以分成三个步骤：

  * 预留：dma_contiguous_reserve()
  * 释放：cma_init_reserved_areas()
  * 分配：cma_alloc()

## 预留

从上面看到dma_contiguous_reserve()的时机在memblock获得了内存布局后，但在sparse_init()之前。

其作用是从memblock中预留连续的空间，并以数据结构cma表示。这样这部分被预留的内存空间，就不会被其他人使用。也不会被直接释放到page allocator。接下来会看到。

## 释放

这里和page allocator一样，**页是先被释放进page allocator，再被分配的**。

为了避免内存浪费，上面被预留的内存会先释放到page allocator，作为普通的页被系统使用。当真正需要用的时候，再替换出来被分配。

## 分配

# 数据结构

```
struct cma cma_areas[MAX_CMA_AREAS];   // 最大支持的cma池子数

cma_declare_contiguous_nid(base, size, limit, alignment, order_per_bit, fixed, name, res_cma, nid)
    cma_init_reserved_mem(base, size, order_per_bit, name, res_cma)

    cma
    +---------------------------+
    |count                      |  number of pages = @size >> PAGE_SHIFT
    |available_count            |  number of pages un-allocated
    |order_per_bit              |  @size should be aligned with order_per_bit
    |flags                      |  state, e.g. CMA_ZONES_VALID/CMA_ACTIVATED
    |                           |
    |nr_ranges                  |  = 1
    |ranges[CMA_MAX_RANGES]     |
    |    (struct cma_memrange)  |
    |    +----------------------+
    |    |base_pfn              |  = PFN_DOWN(base)
    |    |early_pfn             |  = PFN_DOWN(base)
    |    |count                 |  = cma->count
    |    |                      |
    |    |bitmap                |  (count >> order_per_bit) bits
    |    +----------------------+
    |    |                      |
    |    |                      |
    +----+----------------------+
```

# 预留

cma是个有意思的东西。他先要通过memblock将需要的内存区域reserve，这样让别人用不到。然后再把这部分内存释放到buddy系统，好让系统使用。

在使用方需要的时候，再把这部分内存，如果已经被人用的情况下，迁移到空闲内存后，再使用。

## 命令行解析

cma是一个可配置功能，首先编译内核时需要勾选相应的cma配置项。

```
CONFIG_CMA=y
CONFIG_DMA_CMA=y
CONFIG_DEBUG_FS=y
CONFIG_CMA_DEBUGFS=y
```

除此之外，系统启动后是否确实会预留cma，还依赖系统自身的配置，如DTS或者内核参数。

其中一种打开方式是在内核命令行参数中添加cma=xxx的方式

```
cma=256M                 # 全局 256M，地址由内核挑
cma=256M@0x80000000      # 指定起始地址，要对齐 PAGE_SIZE
```

这个部分在内核中early_cma()函数中解析。这个函数就是解析命令行中三个值：

  * size
  * base
  * limit

在后续初始化函数dma_contiguous_reserve()中被使用到。

## 预留过程

```
    dma_contiguous_reserve(max_pfn_mapped << PAGE_SHIFT)  // 给dma分好cma区域,好像把所有的内存都包近来了
        dma_contiguous_reserve_area(, dma_contiguous_default_area, )
            cma_declare_contiguous(base, size, limit, 0, 0, fixed, "reserved", res_cma)
                cma_declare_contiguous_nid() -> __cma_declare_contiguous_nid()
                    cma_alloc_mem(base, size, alignement, limit, nid)
                        base = memblock_alloc_range_nid(size, align, ...)
                    cma_init_reserved_mem(base, size, order_per_bit, name, res_cma)
```

简单来将就是通过memblock_alloc_range_nid()从memblock中reserve了一块空间， 并用cma结构来表示。

# 释放

上面从memblock中预留的空间目前系统中没有人能使用，为了避免浪费，内核会先将这部分内存释放到page allocator，设备需要用的时候再分配。这样避免浪费。

但仔细看了下，这里不完全是释放的操作，在cma能真正被使用前，还需要做一些初始化工作。

```
    cma_init_reserved_areas()
        cma_activate_area(&cma_areas[i])
            cma_validate_zones(cma)                           // 检查空间是否都在一个zone内
            init_cma_reserved_pageblock()                     // 每pageblock设置为MIGRTE_CMA, 并释放到page allocator
                init_pageblock_migratetype(page, MIGRATE_CMA, false)
                set_page_refcounted(page)
                __free_pages(page, pageblock_order)
```

其中cma_init_reserved_areas()会调用init_cma_reserved_pageblock()将page释放到buddy。并且标注为MIGRATE_CMA类型。

# 分配 & 释放

当系统运行后，初始化都完成的情况下，用户就可以像cma申请/归还内存：


  * cma_alloc()
  * cma_release()

## 分配

```
  cma_alloc(cma, count, align, )
      page = cma_alloc_frozen(cma, count, align, ) -> __cma_alloc_frozen(cma, count, align, GFP_KERNEL)
          cma_range_alloc(cma, &cma->ranges[], count, align, &page)
              bitmap_find_next_zero_area_off()                // 找出一段连续的空闲bitmap
              bitmap_set(bitmap, bitmap_no, bitmap_count)     // 标记为已分配


              alloc_contig_frozen_range(pfn, pfn + count, )
      set_pages_refcounted(page, count)                       // 每一个page都设置为1
```

# 例子 & 测试

在内核代码mm/cma_debug.c中，提供了一个测试cma的接口文件。按照前面说的配额制好内核，启动时添加cma=256M参数，我们就可以通过这个接口了解cma的运作。

## 挂载debugfs

```
# 1. 挂载 debugfs
sudo mount -t debugfs none /sys/kernel/debug/

# 2. 进入 cma 调试目录（通常会有一个 cma-0 或以名称命名的目录）
cd /sys/kernel/debug/cma/
ls -l
cd reserved/  # 进入默认 CMA 区域

# 3. 查看当前状态
cat count   # 总页面数
cat used    # 当前被占用的页面数
```

[1]: /mm/02-memblock.md

hugetlbfs是一个伪装成文件系统的内存映射机制，这一点和tmpfs有点像，不过还有它的独特之处。比如，它的初始化流程。

# 总览

```
start_kernel()
    setup_arch()
        parse_early_param()
            parse_early_options()
                parse_args("early options", , do_early_param)      // 只记录命令行参数
                --> default_hugepagesz_setupargs()
                --> hugepagesz_setupargs()
                --> hugepages_setupargs()

    mm_core_init_early()
        sparse_sections_init();
        free_area_init();
        
        hugetlb_cma_reserve();
        hugetlb_bootmem_alloc();
            hugetlb_parse_param();                                 // 真正解析命令行参数
                --> default_hugepagesz_setup()
                --> hugepagesz_setup()
                --> hugepages_setup()

            // for_each_hstate(h), if hstate_is_gigantic(h)        // 先处理超过buddy尺寸的大页
            hugetlb_hstate_alloc_pages(h)                          // 这里只预留了空间
        
        sparse_init();
        memmap_init();

    rest_init()
        kernel_init()
            kernel_init_freeable()
                page_alloc_init_late()
                    hugetlb_bootmem_struct_page_init()
                        gather_bootmem_prealloc_node(nid)          // 初始化大尺寸预留内存
                do_basic_setup()
                    do_initcalls()
                        do_initcall_level()
                            do_one_initcall()
                            --> hugetlb_init()
                                    hugetlb_init_hstates()
                            --> init_hugetlbfs_fs()
```

# 命令行参数处理

hugetlb有三个命令行参数处理函数：

  * default_hugepagesz_setup()
  * hugepagesz_setup()
  * hugepages_setup()

这三个是在系统启动初期，命令行指定默认hugetlb大小和预分配内存大小的。

default_hugepagesz_setup()设置的是默认尺寸，命令行中可以这么写。

> default_hugepagesz=1G

hugepagesz_setup()和hugepages_setup()通常依次组合使用，先设定hstate的尺寸，再设定这个尺寸中要分配的内存大小。

正确的命令行参数写法有：

> hugepagesz=1G hugepages=4 hugepagesz=2M hugepages=1024

甚至可以指定不同numa节点上预留的内存大小

> hugepagesz=1G hugepages=0:4,1:8


```
unsigned int default_hstate_idx;
int hugetlb_max_hstate;                                            // hstates数组中有效个数
struct hstate hstates[HUGE_MAX_HSTATE];
#define default_hstate (hstates[default_hstate_idx])
static struct vfsmount *hugetlbfs_vfsmount[HUGE_MAX_HSTATE];

// 如果命令行只有hugepages的时候，先保存参数，否则会保存到parsed_hstate中
static unsigned long default_hstate_max_huge_pages;
static unsigned int default_hugepages_in_node[MAX_NUMNODES];
static struct hstate * parsed_hstate;
static bool parsed_valid_hugepagesz = true;
static bool parsed_default_hugepagesz;

// 设置默认hugetlb尺寸
default_hugepagesz_setup()
    hugetlb_add_hstate(order)                                      // 添加一个hstate尺寸
        h = &hstates[hugetlb_max_hstate++];
        h->order = order;
        h->mask = ~(huge_page_size(h) - 1);
        parsed_hstate = h;
    parsed_valid_hugepagesz = true;
    parsed_default_hugepagesz = true;
    default_hstate_idx = hstate_index(size_to_hstate(size));

// 指定一个hugetlb尺寸
hugepagesz_setup()
    h = size_to_hstate(size)
    if (h) {
        // 如果h是default_hstate
        parsed_hstate = h;
    }
    hugetlb_add_hstate(order)                                      // 添加一个hstate尺寸
    parsed_valid_hugepagesz = true;

// 根据指定的尺寸，预留大小
hugepages_setup(s)
    if (!parsed_valid_hugepagesz) return;                          // 必须已经设置过尺寸

    // 解析命令行数值，保存到mhp中
```

到这里，我们还没有看到真实的内存预留，只是把用户参数保存起来。

# hstate

hstate是hugetlb核心数据结构，对应每一个支持的hugetlb尺寸都有一个hstate结构。

hstate第一次创建由hugetlb_add_hstate()完成。

```
    hstate
    +--------------------------------------------+
    |name[]                                      | = hugepages-xxkB
    |                                            |
    |order                                       | 尺寸的order
    |mask                                        | 尺寸大小的mask
    |demote_order                                |
    |                                            |
    |max_huge_pages                              | 预留总页数
    |max_huge_pages_node[]                       | 每个节点上预留总页数
    |                                            |
    |nr_huge_pages                               |
    |nr_huge_pages_node[]                        |
    |                                            |
    |free_huge_pages                             |
    |free_huge_pages_node[]                      |
    |                                            |
    |hugepage_freelists[]                        |
    |hugepage_activelist                         |
    |    (struct list_head)                      |
    +--------------------------------------------+
```

# hugetlb_init()

```
static int num_fault_mutexes __ro_after_init;
struct mutex *hugetlb_fault_mutex_table __ro_after_init;

hugetlb_init()
    hugetlb_add_hstate(HUGETLB_PAGE_ORDER)                         // 即使命令行没有指定，也会支持

    hugetlb_init_hstates()
        // for_each_hstate(h)
        hugetlb_hstate_alloc_pages(h)                              // 非gigantic的在这里分配内存

    // 打印日志： HugeTLB: registered %s page size, pre-allocated %ld pages
    report_hugepages()

    hugetlb_sysfs_init()
        // for_each_hstate(h)
        hugetlb_sysfs_add_hstate()                                 // 目录/sys/kernel/mm/hugepages/

    hugetlb_cgroup_file_init()                                     // 目录/sys/fs/cgroup/hugetlb/
    hugetlb_sysctl_init()                                          // 目录/proc/sys/vm/ (以sysfs为主)

    // 初始化互斥锁
    num_fault_mutexes = roundup_pow_of_two(8 * num_possible_cpus());
    hugetlb_fault_mutex_table =
        kmalloc_objs(struct mutex, num_fault_mutexes);
    for (i = 0; i < num_fault_mutexes; i++)
        mutex_init(&hugetlb_fault_mutex_table[i]);
```

## hugetlb_hstate_alloc_pages()

该函数用来预留hugetlb所需内存，如注释中所说，分两种情况：

  * gigantic
  * non-gigantic

前者在内存还在memblock的时候就分配了，这样保证内存连续。后者则是在内存初始化完，从buddy中分配。

```
hugetlb_hstate_alloc_pages(h)
    hugetlb_hstate_alloc_pages_specific_nodes(h)
        hugetlb_hstate_alloc_pages_onenode(h)

    // 启动时预留大尺寸
    hugetlb_gigantic_pages_alloc_boot(h)

    // 启动后，预留内存
    hugetlb_pages_alloc_boot(h)

    hugetlb_hstate_alloc_pages_errcheck()
```

针对单node的分配好像不常用，我们先跳过。所以关注点就是

  * 大尺寸： hugetlb_gigantic_pages_alloc_boot()
  * 正常尺寸： hugetlb_pages_alloc_boot()

大尺寸hugetlb的内存预留

```
struct list_head huge_boot_pages[MAX_NUMNODES];                    // 预留的内存挂载在链表上

hugetlb_gigantic_pages_alloc_boot(h)
    // for (i = 0; i < h->max_huge_pages; ++i), 寸的是页数，所以一个计数就分配一次
    alloc_bootmem_huge_page(h, NUMA_NO_NODE)
        m = arch_alloc_bootmem_huge_page(h, nid)
            alloc_bootmem(h, nid, false)
                __memblock_reserve(addr, size)
                memblock_prep_allocation(addr, size, true)
                return phys_to_virt(addr)

        list_add_tail(&m->list, &huge_boot_pages[nid])
        m->flags |= HUGE_BOOTMEM_ZONES_VALID;
        hugetlb_vmemmap_optimize_bootmem_page(pfn, huge_page_order(h))
        // 标注第一个page不需要初始化, 在memmap_init_reserved_pages中跳过
        memblock_reserved_mark_noinit()

    // 如果预留的内存区域跨zone， 释放
    hugetlb_free_cross_zone_pages(h, nid)


```

# init_hugetlbfs_fs()

页结构体(struct page)可以说是我见过的最为繁杂的结构体定义了，至今我只能说是略懂。然而这个结构体定义又是那么的重要，因为物理意义上的每个页面都对应一个结构体。可以说这个结构体是整个系统上所有内存的守护者。

每个细分成员的作用就不在本节展开了，这里主要是作为一个全局参考，从整体上对页结构体有个了解。具体的作用将在对应的章节中描述。

# 庞然大物

```
    struct page (include/linux/mm_types.h)

                         page
               +--------------------------------------------------------------+
               |flags                                                         |    page-flags-layout.h
               |   (unsigned long)                                            |    page-flags.h
   --+--       +==============================================================+
     |         |..............................................................|
     |         |page cache and anonymous pages                                |
     |         |    +---------------------------------------------------------+
     |         |    |lru                                                      |
     |         |    |    (struct list_head)                                   |
               |    |mapping                                                  |
               |    |    (struct address_space*)                              |
               |    |index                                                    |
               |    |    (pgoff_t)                                            |
  5 words      |    |private                                                  |
   union       |    |    (unsigned long)                                      |
               |    +---------------------------------------------------------+
               |..............................................................|
    has        |slab, slob, slub                                              |
               |    +---------------------------------------------------------+
  7 usage      |    |.........................................................|
               |    |                            .+---------------------------|
               |    |slab_list                   .|next                       |
               |    |    (struct list_head)      .|   (struct page*)          |
               |    |                            .|pages                      |
               |    |                            .|pobjects                   |
               |    |                            .|   (int)                   |
               |    |                            .+---------------------------|
               |    |.........................................................|
               |    +---------------------------------------------------------+
               |    |slab_cache                                               |
               |    |    (struct kmem_cache*)                                 |
               |    |freelist                                                 |
               |    |    (void*)                                              |
               |    +---------------------------------------------------------+
               |    |.........................................................|
               |    |s_mem    .counters          .+---------------------------|
               |    | (void*) . (unsigned long)  .|inuse                      |
               |    |         .                  .|objects                    |
               |    |         .                  .|frozen                     |
               |    |         .                  .|    (unsigned)             |
               |    |         .                  .+---------------------------|
               |    |.........................................................|
               |    +---------------------------------------------------------+
               |..............................................................|
               |Tail pages of compound page                                   |
               |    +---------------------------------------------------------+
               |    |compound_head                                            |
               |    |    (unsigned long)                                      |
               |    |compound_dtor                                            |
               |    |compound_order                                           |
               |    |    (unsigned char)                                      |
               |    |compound_mapcount                                        |
               |    |    (atomic_t)                                           |
               |    +---------------------------------------------------------+
               |..............................................................|
               |Second tail page of compound page                             |
               |    +---------------------------------------------------------+
               |    |_compound_pad_1                                          |
               |    |_compound_pad_2                                          |
               |    |    (unsigned long)                                      |
               |    |deferred_list                                            |
               |    |    (struct list_head)                                   |
               |    +---------------------------------------------------------+
               |..............................................................|
               |Page table pages                                              |
               |    +---------------------------------------------------------+
               |    |_pt_pad_1                                                |
               |    |    (unsigned long)                                      |
               |    |pmd_huge_pte                                             |
               |    |    (pgtable_t)                                          |
               |    |_pt_pad_2                                                |
               |    |    (unsigned long)                                      |
               |    |.........................................................|
               |    |pt_mm                         .pt_frag_refcount          |
               |    |    (struct mm_struct*)       .    (atomic_t)            |
               |    |.........................................................|
               |    |ptl                                                      |
               |    |    (spinlock_t/spinlock_t *)                            |
               |    +---------------------------------------------------------+
               |..............................................................|
               |ZONE_DEVICE pages                                             |
               |    +---------------------------------------------------------+
               |    |pgmap                                                    |
               |    |    (struct dev_pagemap*)                                |
               |    |hmm_data                                                 |
               |    |_zd_pad_1                                                |
     |         |    |    (unsigned long)                                      |
     |         |    +---------------------------------------------------------+
     |         |..............................................................|
     |         |rcu_head                                                      |
     |         |    (struct rcu_head)                                         |
     |         |..............................................................|
   --+--       +==============================================================+
     |         |..............................................................|
               |            .                 .                .              |
   4 bytes     |_mapcount   .page_type        .active          .units         |
    union      |  (atomic_t).   (unsigned int).  (unsigned int).   (int)      |
               |            .                 .                .              |
     |         |..............................................................|
   --+--       +==============================================================+
               |_refcount                                                     |
               |     (atomic_t)                                               |
               |mem_cgroup                                                    |
               |     (struct mem_cgroup)                                      |
               |virtual                                                       |
               |     (void *)                                                 |
               |_last_cpupid                                                  |
               |     (int)                                                    |
               +--------------------------------------------------------------+
```

# Flags

在页结构体众多成员中，flasgs字段在任何用途下都标示了对应页的属性。除此之外内核中还对这个字段做了非常巧（变）妙（态）的布局。

内核中有两个头文件和flags的定义有着密切的联系

    * include/linux/page-flags-layout.h
    * include/linux/page-flags.h

前者定义了该字段的布局，后者则定义了该字段具体的意义和相关操作的宏定义。

## 布局

以下对字段布局的描述摘自page-flags-layout.h

```
/*
 * page->flags layout:
 *
 * There are five possibilities for how page->flags get laid out.  The first
 * pair is for the normal case without sparsemem. The second pair is for
 * sparsemem when there is plenty of space for node and section information.
 * The last is when there is insufficient space in page->flags and a separate
 * lookup is necessary.
 *
 * No sparsemem or sparsemem vmemmap: |       NODE     | ZONE |             ... | FLAGS |
 *      " plus space for last_cpupid: |       NODE     | ZONE | LAST_CPUPID ... | FLAGS |
 * classic sparse with space for node:| SECTION | NODE | ZONE |             ... | FLAGS |
 *      " plus space for last_cpupid: | SECTION | NODE | ZONE | LAST_CPUPID ... | FLAGS |
 * classic sparse no space for node:  | SECTION |     ZONE    | ... | FLAGS |
 */
```

# 操作

而具体字段的含义实在太过复杂，大家可以直接查看源代码。其中对字段操作的宏定义总结如下：

    * PageXXX()
    * SetPageXXX()
    * ClearPage()

这三个宏分别用来判断、设置、清楚页结构体中相应的属性。在内核代码中使用非常广泛，以后看到长成这样的，就可以到这个头文件中搜索。

内存从页分配器（buddy系统）中分配出去后，就散落在了系统的各个角落。对于一个特定的pfn或者page，怎么知道这个页是分配给了谁，用在了哪里是内核对内存管理的一个任务。

比如函数memory_failure中就对不同用途的页做了不同的出错处理。下面我将列出我所知道的页分配器的“知名”用户们。

对于页用途的标示，在内核中又分成两类途径：

  * page->flags
  * page->page_type

不知道为什么要分成两个字段来做区分，可能是用满了吧。

鉴于用page_type来区分的类型较少，我们就先介绍page_type中的类型。

# 以page_type分类的类型

page_type支持的类型一共就五种

```
#define PG_buddy	0x00000080
#define PG_offline	0x00000100
#define PG_kmemcg	0x00000200
#define PG_table	0x00000400
#define PG_guard	0x00000800
```

对应的操作通过PAGE_TYPE_OPS来定义，在使用中也是用__SetPageXXX和__ClearPageXXX来判断。

## PageTable

Marks pages in use as page tables. 也就是这个页我们是用来做页表的。

设置这个类型的地方只有天字唯一一个地方：

```
__pte_alloc_one
    pgtable_pte_page_ctor
        __SetPageTable(page)
```

因为页表，至少在x86上占一个页面，所以没有compound page的可能。

## PageBuddy

这个用来判断一个页是是否在buddy系统中，如果在的话，说明还没有分配出去。

但是有意思的是，并不是所有在buddy系统中的空闲页都会设置这个类型。而是只有在整个页的page[0]对应的page_type才会被设置。所以函数is_free_buddy_page()就这样诞生了。

设置这个类型的地方也是只有天字唯一一个地方：

```
__free_one_page()
    set_page_order()
        set_page_private(page, order);
  	    __SetPageBuddy(page);
```

因为只有对头部的page设置这个值，所以这个判断的作用需要进一步考量。

## PageGuard

这是一个在debug情况下才用到的判断。

调用的地方也只有一个：

```
expand()
    set_page_guard()
        __SetPageGuard()
```

这样被分片的页就不能再被分配了。也不知道这是什么调试技巧。

# 以flags分类的类型

## PageHuge

这个判断用来确认对应的页是不是hugetlbfs中分配的。其中有个重要的依据就是compound_dtor是不是HUGETLB_PAGE_DTOR。

这个设置的工作在prep_new_huge_page()函数中完成。

从PageHuge的代码中可以看到，被判断的为真的页必须是compound page。这个设置的过程有点复杂。

比如在alloc_fresh_huge_page()函数中，分成两种情况。

  * alloc_gigantic_page
  * alloc_buddy_huge_page

后者alloc_buddy_huge_page函数中，明确设置了__GFP_COMP。
但是在alloc_gigantic_page中，并没有用通用的方式设置compound page。而是在prep_compound_gigantic_page中设置了对应的Head/Tail。这一点颇为有趣。

## PageLRU

这个判断用来标示指定的页是不是在lru链表上。在pagevec cache上的页，也不会设置这个标示。

咨询了一下huangying，一共有四种情况会把page放到lru链表上。

   * anonymous page
   * file backend pages
   * page cache(read/write through syscall)
   * shmem

按照我的理解，添加到lru链表的一个重要来源如下：

```
__lru_cache_add()
    __pagevec_lru_add()
        __pagevec_lru_add_fn()
```

而__lru_cache_add()函数的调用者有三种情况：

   * lru_cache_add_anon
   * lru_cache_add_file
   * lru_cache_add

调用的地方较多，其中有一个地方的流程是：

```
handle_pte_fault
    do_anonymous_page
        page = alloc_zeroed_user_highpage_movable()
        lru_cache_add_active_or_unevictable()
            lru_cache_add()
```

从这一点上至少可以证明普通进程中的映射的页会添加到lru链表中。同样看了一眼hugepage的流程，hugepage的页面也会添加到lru链表中。

## PageSlab()

对于给slab分配器使用的内存，只要这个用来判断就知道了。

设置这个bit的位置只有一个，在函数allocate_slab()中。从页分配器中获得了page后，则会做上相应的标记。

另外需要说明的是，slab中分配到的页只要是高阶的那就是compound page。因为在函数calculate_size中会做如下操作。

```
if (order)
  s->allocflags |= __GFP_COMP;
```

## PageTransHuge

这个函数说是用来判断Transparent Huge Page的，但说实话我没有看懂它是怎么做到的。因为它其实只是判断了是不是PageHead。

那我们还是来看看Transparent Huge Page是怎么个分配的吧。

Transparent Huge Page的分配发生在缺页中断，其中有两个地方可能分配create_huge_pud和create_huge_pmd。

我们以匿名页表的pmd创建为例。

```
do_huge_pmd_anonymous_page
    gfp = alloc_hugepage_direct_gfpmask(vma);
        return GFP_TRANSHUGE or GFP_TRANSHUGE_LIGHT
    page = alloc_hugepage_vma(gfp, vma, haddr, HPAGE_PMD_ORDER);
    prep_transhuge_page(page);
        set_compound_page_dtor(page, TRANSHUGE_PAGE_DTOR)
```

其中值得注意的是gfp的计算和prep_transhuge_page。

alloc_hugepage_direct_gfpmask返回的gfp都包含了__GFP_COMP。这一点保证了得到的页一定是compound page。
prep_transhuge_page中设置了compound_dtor。个人感觉用这个来做判断更加准确。

# PageSwapCache

有几个地方设置，其中之一在函数add_to_swap_cache()中。

```
  shrink_page_list
    add_to_swap(page)
      add_to_swap_cache(page, entry, gfp)
        SetPageSwapCache(page)
```

感觉这条线还是从kswapd下来的。

# PageSwapBacked

这个也有几个地方可以设置，其中一个在缺页中断的do_swap_page()。

```
  handle_pte_fault
    if (!pte_present(vmf->orig_pte))
      return do_swap_page(vmf);
        __SetPageSwapBacked(page)
```

感觉意思是发生缺页中断，如果这个页有在swap中了，那就标记这个位。

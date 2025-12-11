进入细节前，我们先来看看有哪几种情况会触发拆分。

# 手动拆分

为了测试split的功能，内核还提供一个调试接口文件/sys/kernel/debug/split_huge_pages。按照格式写入该文件，则会触发对应页面的split。

PS: 也可以参考tools/testing/selftests/mm/split_huge_page_test.c中的使用方法。

## 使用方法

这个文件在mm/huge_memory.c中创建的，只有写的接口，没有读的。对应的方法是split_huge_pages_write()。

写这个文件分为两种格式： 文件和进程的。这次我们只看进程的。

在split_huge_page_test.c中有定义这个格式：

```
#define PID_FMT "%d,0x%lx,0x%lx,%d"
```

也就是 pid, start_vaddr, end_vaddr, new_order

可以看到，因为现在有mTHP了，所以拆分的时候可以指定拆分到的页面大小。当new_order省略时，默认位0。

# 自动拆分--deferred_split_shrinker

系统运行时一般情况下我们不会去手动拆分大页，而是通过shrinker扫描将浪费的大页拆分 -- deferred_split_shrinker。

shrinker的机制不在本章范围，简单来说函数do_shrink_slab()会在必要的时候调用count_objects/scan_objects。而deferred_split_shrinker的这两个成员是：

  * deferred_split_count
  * deferred_split_scan

前者就是查看对应ds_queue->split_queuue_len，后者则需要执行关键的操作split_folio()。

## 添加到deferred_list

首先当分配pmd folio的时候，会调用deferred_split_folio()将folio添加到对应的deferred_list中。

这样在deferred_split_scan的时候才会扫描到对应的folio，然后继续做拆分。

## 判断是否需要拆分

这个任务交给了thp_underused()，就是判断空闲的页面是否达到一定的数量。

但是这只针对没有partially mapped的folio，如果是partially mapped的一定会拆分。

## 拆分

最后重要的步骤就是拆分 -- split_folio()

不过最后看下来，核心的函数是__folio_split()。内核里这名字真的都是艺术。

# 拆分的细节

## 整体流程

按照目前理解，整理一下__folio_split()的流程。

```
    folio_trylock(folio)
    folio_try_get(folio)                     这两个是在进入__folio_split()前做的
    __folio_split()
    
        unmap_folio(folio)                       拆分页表

        folio_ref_freeze(folio, 1 + extra_pins)  正是因为unmap了，所以这个计数是这样
        __split_unmapped_folio()
        folio_ref_unfreeze()                     unfreeze拆分后的所有folio
    
        remap_page()                             重新映射页表
```

## __folio_split()碎碎念

在阅读__folio_split()时，发现有几个有意思的点，在这里单独列出来算是一些知识点的补充。

1. can_split_folio()

   从这个函数可以看到folio_mapcount()和folio_ref_count()之间的关系。

2. folio_try_get()和folio_ref_freeze()

   我都一次知道freeze的意思是把refcount设置为0,这样其他人在folio_try_get()的时候就会失败。

## __split_unmapped_folio()

真正拆分folio的活落在了这个函数上。

## 拆分页表

仅仅拆分大页，工作还不算做完。因为对应的页表也需要拆分，否则如果大页中的部分被释放了，那么系统就会访问不该访问的区域。

这部分工作藏得有点深，结合在unmap的步骤中。

```
    __folio_split
        unmap_folio(folio)
            try_to_unmap / try_to_migrate
                rmap_walk_locked
                    try_to_migrate_one / try_to_unmap_one
                        ...
                        split_huge_pmd_locked
        remap_page
```

具体工作由函数__split_huge_pmd_locked完成。

```
__split_huge_pmd_locked
    pgtable = pgtable_trans_huge_withdraw(mm, pmd);
    pmd_populate(mm, &_pmd, pgtable);
    for (i = 0; i < HPAGE_PMD_NR; i++) {
        set_pte_at(mm, addr, pte, entry);
    }
    pmd_populate(mm, pmd, pgtable);
```

其实说白了很简单，就是把预留了页表拿出来，每一项都填上大页对应的地址。最后把相应的pmd改好。是不是有点酷。

但是如果这里就把pte都映射好了，那为什么还要在remap_page()呢？

# 测试

针对大页拆分，内核中提供了一个用户态的测试程序。

tools/testing/selftests/mm/split_huge_page_test.c

这算是[selftests][1]的一个例子。

[1]: /mm/tests/01_functional_test.md

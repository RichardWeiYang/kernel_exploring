首先是THP的分配过程，也就是在缺页中断中处理的。

```
handle_mm_fault()
    hugetlb_fault
    __handle_mm_fault
        create_huge_pmd()
        do_huge_pmd_numa_page
        wp_huge_pmd
```

如果/sys/kernel/mm/transparent_hugepage/有全局配置，或者vma中标识了使用VM_HUGEPAGE。在发生缺页中断时，则会分配大页。

# 预留页表

在创建大页表项的时候，与其他人不同，还预留了一张页表。分别对应两个函数。

* pgtable_trans_huge_deposit()
* pgtable_trans_huge_withdraw()

我们先来看第一个。这个函数的动作是预留了一个页表。那什么时候用呢？就是在第二个函数发生的时候。那为什么要这么做呢？这个就牵扯到下一个不同点--拆分页表。

我们知道对于大页，页表一共有三级。而正常的4k页的页表是四级。所以当我们要拆分大页页表时，就需要补上这么一级。为了保证拆分时，不会因为内存不够导致不能展开到四级页表，所以在分配时就多预留了一个页表。

不得不说这是一个有意思的设计。


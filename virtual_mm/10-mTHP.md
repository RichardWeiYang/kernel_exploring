mTHP是对THP的进一步优化，作者是Ryan在2023年推进社区的。

下面是相关的patch

```
c0f79103322c 2023-12-20 selftests/mm/cow: add tests for anonymous multi-size THP
12dc16b38463 2023-12-20 selftests/mm/cow: generalize do_run_with_thp() helper
9f0704eae8a4 2023-12-20 selftests/mm/khugepaged: enlighten for multi-size THP
4f5070a5e40d 2023-12-20 selftests/mm: support multi-size THP interface in thp_settings
00679a183ac6 2023-12-20 selftests/mm: factor out thp settings management
b6aab3384caf 2023-12-20 selftests/mm/kugepaged: restore thp settings at exit
19eaf44954df 2023-12-20 mm: thp: support allocation of anonymous multi-size THP
3485b88390b0 2023-12-20 mm: thp: introduce multi-size THP sysfs interface
372cbd4d5a06 2023-12-20 mm: non-pmd-mappable, large folios for folio_add_new_anon_rmap()
7dc7c5ef6463 2023-12-20 mm: allow deferred splitting of arbitrary anon large folios
```

mTHP和THP不一样的点在与，THP是真的在物理上把512个PTE压缩成了一个pmd。而mTHP是在缺页中断的时候，根据配置一次性设置2^order个pte，从而达到减少缺页中断，计数更改操作的效果。

# 状态统计

在/sys目录下，还增加了mTHP相关的状态统计值，用来观察mTHP的过程。

比如在目录/sys/kernel/mm/transparent_hugepage/hugepages-128kB/stats/中有

```
$ls
anon_fault_alloc	    shmem_fallback	   swpin_fallback
anon_fault_fallback	    shmem_fallback_charge  swpin_fallback_charge
anon_fault_fallback_charge  split		   swpout
nr_anon			    split_deferred	   swpout_fallback
nr_anon_partially_mapped    split_failed	   zswpout
shmem_alloc		    swpin
```

具体含义可以在Documentation/admin-guide/mm/transhuge.rst文档中查看。

不过在/proc/vmstat中也有一些和thp相关的计数。都是以thp_开头的。也可以在transhuge.rst文档中查看。

# 分配过程

目前mTHP分配发生在缺页中断中：

```
handle_pte_fault()
    do_pte_missing()
        do_anonymous_page()
            alloc_anon_folio()                                         // 去判断是否有合适的mTHP满足
            folio_ref_add(folio, nr_pages - 1)                         // 计数设置为nr_pages
            folio_add_new_anon_rmap(folio, vma, addr, RMAP_EXCLUSIVE)  // 调整mapcount
```

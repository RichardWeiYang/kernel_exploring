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


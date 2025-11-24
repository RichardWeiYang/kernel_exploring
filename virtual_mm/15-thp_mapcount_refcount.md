之前我们在[mapcount][1]中研究过Folio的mapcount变化情况，但是当时我们并没有具体的例子。

现在我们结合thp的分配/拆分/合并，来展开看看mapcount在这些过程中是如何变化的。

与此同时，folio中的refcount和mapcount还有一定的关系。比如在folio_expected_ref_count()中，其中就考虑到了两者的关系。

```
static inline int folio_expected_ref_count(const struct folio *folio)
{
	const int order = folio_order(folio);
	int ref_count = 0;

	...

	/* One reference per page table mapping. */
	return ref_count + folio_mapcount(folio);
}
```

我们只看最后一行的注释，这说明只要folio在页表中被映射一条，那就会拿到一个refcount。

所以事实是不是如此呢？

# THP 分配

```
    do_huge_pmd_anonymous_page -> __do_huge_pmd_anonymous_page
        vma_alloc_anon_folio_pmd() -> vma_alloc_folio()
            folio_alloc_noprof() -> __folio_alloc_noprof()
                set_page_refcounted(page)                                // refcount = 1
        map_anon_folio_pmd_pf() -> map_anon_folio_pmd_nopf()
            folio_add_new_anon_rmap(folio, vma, addr, RMAP_EXCLUSIVE)
```

设置后的结果：

```
_entire_mapcount: 1
_large_mapcount:  1
_nr_pages_mapped: ENTIRELY_MAPPED

_refcount:        1
```

此时我们可以看到 _refcount == _large_mapcount，这个是符合预期的。

## mTHP 分配

```
    do_anonymous_page
        alloc_anon_folio() -> vma_alloc_folio()
            folio_alloc_noprof() -> __folio_alloc_noprof()
                set_page_refcounted(page)                                // refcount = 1
        folio_ref_add(folio, nr_pages - 1)                               // refcount = nr_pages
        folio_add_new_anon_rmap(folio, vma, addr, RMAP_EXCLUSIVE)
```

设置后的结果：

```
_mapcount:        1
_large_mapcount:  nr_pages
_nr_pages_mapped: nr_pages

_refcount:        nr_pages
```

一开始我有个疑问，为什么这种情况下_large_mapcount还要加nr_pages，直接加1不好么？

后来想想是有道理的。

  * 从语义上讲，一个_large_mapcount计数代表了有多少个PTE/PMD映射了这个folio
  * 从代码上看，其中要求一个mapcount对应一个refcount

所以这个时候也是符合预期的， _refcount == _large_mapcount。

# THP 拆分

在[THP split][2]中，我们学习了一下大页拆分的过程。

其中涉及到mapcount和refcount的有三个地方：

  * unmap_folio()
  * __folio_freeze_and_split_unmapped()
  * remap_page()

按照目前的理解，unmap_folio()执行后，folio中mapcount有关的计数应当都清零了。

下面我们以匿名页为例分析一下。

## unmap_folio

这个执行的流程在：

```
__folio_split
    // folio_expected_ref_count(folio) == folio_ref_count(folio) - 1

    try_to_migrate_one
        split_huge_pmd_locked
            folio_remove_rmap_pmd
                __folio_remove_rmap(, PGTABLE_LEVEL_PMD)
            put_page()

```

首先明确一点能够拆分的前提需要folio_expected_ref_count()和folio_ref_count()差1。（因为split前增加了1）

对于不在swapcache中的匿名页，此时refcount正好等于mapcount。

设置的变动如下： (-1表示原值基础上-1)

```
_entire_mapcount: -1
_large_mapcount:  -1
_nr_pages_mapped: -ENTIRELY_MAPPED

_refcount:        -1
```

这么看，这样正好和之前分配大页时的操作对应。把分配时设置的mapcount都清除了。

然后refcount也减1,这样就剩下为了__folio_split()额外增加的1个计数。

## __folio_freeze_and_split_unmapped

这个函数主要将folio拆分，并处理新的folio上的refcount。其中重要的两个地方是：

  * folio_ref_freeze()
  * folio_ref_unfreeze()

经过unmap_folio()，现在folio上的refcount应当只有pagecache和swapcache上的计数。如果freeze不成功，说明此时还有人在使用这个folio，所以不能继续。

而对于拆分后的新folio，通过计算出folio_expected_ref_count()后，将这个值加1后folio_ref_unfreeze()。

值的一提的是pagecache folio其实一直处于mapping状态。这个地方是我目前不清楚的，需要后面补上。

拆分后的不在swapcache中的匿名folio设置的变动如下： (-1表示原值基础上-1)

```
_entire_mapcount:  0
_large_mapcount:   0
_nr_pages_mapped:  0

_refcount:         0
```

## remap_folio

这个执行的流程在：

```
    remove_migration_pte
        folio_get(folio)
        folio_add_anon_rmap_pte
            __folio_add_anon_rmap(, PGTABLE_LEVEL_PTE)
```

有意思的是remove_migration_pte每次只map一个pte。

拆分后的不在swapcache中的匿名folio设置的变动如下： (-1表示原值基础上-1)

```
_entire_mapcount:  0
_large_mapcount:   +nr_pages
_nr_pages_mapped:  +nr_pages

_refcount:         +nr_pages
```

这个结果和mTHP初次分配映射的情况对应上了。

# THP 合并

[1]: /virtual_mm/09-mapcount.md
[2]: /virtual_mm/13-thp_split.md

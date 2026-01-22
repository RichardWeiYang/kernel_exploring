之前我们在[mapcount][1]中研究过Folio的mapcount变化情况，但是当时我们并没有具体的例子。

现在我们结合thp的分配/拆分/合并，来展开看看mapcount在这些过程中是如何变化的。

与此同时，folio中的refcount和mapcount还有一定的关系。比如在folio_expected_ref_count()中，其中就考虑到了两者的关系。

```
static inline int folio_expected_ref_count(const struct folio *folio)
{
	const int order = folio_order(folio);
	int ref_count = 0;

	/* One reference per page from the swapcache. */
	ref_count += folio_test_swapcache(folio) << order;

	if (!folio_test_anon(folio)) {
		/* One reference per page from the pagecache. */
		ref_count += !!folio->mapping << order;
		/* One reference from PG_private. */
		ref_count += folio_test_private(folio);
	}

	/* One reference per page table mapping. */
	return ref_count + folio_mapcount(folio);
}
```

我们只看最后一行的注释，这说明只要folio在页表中被映射一条，那就会拿到一个refcount。

**强调一点，如果一个匿名页没有保存到swapcache中，那么 refcount == mapcount**

所以事实是不是如此呢？接下来我们分别从分配、拆分和合并这三个过程来验证。

# THP 分配

```
    do_huge_pmd_anonymous_page -> __do_huge_pmd_anonymous_page
        vma_alloc_anon_folio_pmd() -> vma_alloc_folio()
            folio_alloc_noprof() -> __folio_alloc_noprof()
                set_page_refcounted(page)                                // refcount = 1
        map_anon_folio_pmd_pf() -> map_anon_folio_pmd_nopf()
            folio_add_new_anon_rmap(folio, vma, addr, RMAP_EXCLUSIVE)
                folio_set_large_mapcount(folio, 1, vma)                  // large_mapcount = 1
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
            folio_set_large_mapcount(folio, nr, vma)                     // large_mapcount = nr_pages
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

整个过程中，mapcount和refcount经历了几次调整。其中涉及到mapcount和refcount的有四个地方：

  * unmap_folio(): 删除页表映射
  * __folio_freeze_and_split_unmapped(): freeze大页，并unfreeze拆分后的小页
  * remap_page():  重新映射页表
  * free_folio_and_swap_cache(): 释放freeze时额外的计数

下面我们以没有添加到swapcache中的匿名页为例分析一下。

## unmap_folio()

对于匿名页，这个执行的流程是try_to_migrate()。我们观察一个pmd大页拆分的流程：

```
__folio_split()
    // folio_expected_ref_count(folio) == folio_ref_count(folio) - 1

    unmap_folio()
        try_to_migrate() -> try_to_migrate_one()
            split_huge_pmd_locked()
                folio_remove_rmap_pmd()
                    __folio_remove_rmap(, PGTABLE_LEVEL_PMD)
                        folio_dec_large_mapcount(folio, vma)         // large_mapcoun -= 1;
                folio_put()                                          // refcount -= 1
```

首先明确一点能够拆分的前提需要folio_expected_ref_count()和folio_ref_count()差1。（因为split前增加了1）

对于不在swapcache中的匿名页，正常情况下refcount正好等于mapcount。又因为split前额外获得了一个引用计数，所以 folio_ref_count(folio) == folio_expected_ref_count(folio) + 1 才符合拆分的条件，可以往下继续进行。

设置的变动如下： (-1表示原值基础上-1)

```
_entire_mapcount: -1
_large_mapcount:  -1
_nr_pages_mapped: -ENTIRELY_MAPPED

_refcount:        -1
```

这么看，这样正好和之前分配大页时的操作对应。把分配时设置的mapcount都清除了。

然后refcount也减1,这样就剩下为了**__folio_split()额外增加的1个计数**。

## __folio_freeze_and_split_unmapped

这个函数主要将folio拆分，并处理新的folio上的refcount。其中重要的两个地方是：

  * folio_ref_freeze(folio, folio_cache_ref_count(folio) + 1)
  * folio_ref_unfreeze(new_folio, folio_cache_ref_count(new_folio) + 1)

经过unmap_folio()，现在folio上的refcount应当只有pagecache和swapcache上的计数。如果freeze不成功，说明此时还有人在使用这个folio，所以不能继续。

而对于拆分后的新folio，通过计算出folio_expected_ref_count()后，将这个值加1后folio_ref_unfreeze()。

值的一提的是pagecache folio其实一直处于mapping状态。这个地方是我目前不清楚的，需要后面补上。

拆分后的不在swapcache中的匿名folio设置的变动如下： (-1表示原值基础上-1)

```
_entire_mapcount:  0
_large_mapcount:   0
_nr_pages_mapped:  0

_refcount:        +1
```

对于不在swapcache中的匿名页，在没有映射的情况下，mapcount和refcount都是0。所以folio_ref_freeze()和folio_ref_unfreeze()都是将refcount设置为1，表示为刚好有一个用户在使用(也就是我们这个正在拆分大页的用户)。

但是这个和系统中正常时期望的refcount不一致，正常情况下没有额外用户且没有被映射的匿名页refcount应当是0。那接下来的问题就是，什么时候释放这个额外的引用计数呢？不急，往下看。

## remap_folio()

对于匿名页，remap的过程实际是一个恢复migration entry的过程:

```
remap_page()
    remove_migration_ptes() -> remove_migration_pte()
        folio_get(folio)                                         // refcount += 1
        folio_add_anon_rmap_pte
            __folio_add_anon_rmap(, PGTABLE_LEVEL_PTE)           // large_mapcoun += 1
```

有意思的是remove_migration_pte每次只map一个pte，所以如果整个folio都有被映射的话，那么数值增加nr_pages。

重新映射不在swapcache中的匿名folio的变动如下： (-1表示原值基础上-1)

```
_entire_mapcount:  0
_large_mapcount:   +nr_pages
_nr_pages_mapped:  +nr_pages

_refcount:         +nr_pages
```

这个结果和mTHP初次分配映射的情况对应上了。

到这里，这个匿名页的refcount是不是又和mapcount一致了？ 别急，还记得我们刚才额外增加的存在计数吗？ 所以此时的refcount应当是nr_pages + 1。 这个额外的计数还没有处理掉。

## free_folio_and_swap_cache()

```
#define free_folio_and_swap_cache(folio) \
	folio_put(folio)
```

这个函数简单情况下直接定义为了folio_put()。这样就把刚才额外的存在计数给清除了。

如果这个folio没有别任何进程映射，则此时就会被释放。终于，到这里整个mapcoun/refcount在拆分流程中的变化算整理清除了。

PS: 对@lock_at锁在的folio则会跳过这个操作，这样调用者就能继续处理这个folio了。

## lru_add_split_folio()

到上面为止，基本上把整个大页拆分过程中mapcount/refcount的变化梳理清除了。不过后来又发现一个程咬金，在lru_add_split_folio()中，还会folio_get()一次。

这个情况略微有点特殊，也就是**当split中有一个list参数的时候**，对拆分下来的新页会额外再增加一个计数。这样这个新拆分的folio就能够被添加到原有的处理逻辑，作为下一次继续处理的单元。（因为原先加到list的folio，也是增加了计数后添加到链表里的）

# THP 合并

[1]: /virtual_mm/09-mapcount.md
[2]: /virtual_mm/13-thp_split.md

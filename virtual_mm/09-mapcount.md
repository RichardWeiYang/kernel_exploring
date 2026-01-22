当分配内存给进程使用，被映射到进程页表中时，内核会在page中记录page被映射的次数。也是因为有大页的存在，这个计数的过程变得有点复杂。

David清理的[文章][2]

# 计数都放在哪里

既然是要计数，那就一定有记录的位置。本质上计数还是放在struct
page中，在这里我们把相关的内容抽出来看，更清楚理解其中的使用方式。毕竟page太[让人眼花缭乱了][1]。

现在有两种表达page的结构，struct page和struct folio。为了清楚，下面分别展示一下计数的位置，其中每一行代表一个word。

```
  page
  +----------------------------------+
  |flags                             |
  |         compound_head            |
  |                                  |
  |mapping                           |
  |share                             |
  |private                           |
  |_mapcount/page_type               |
  |_refcount                         |
  |memcg_data / _unused_slab_obj_exts|
  |virtual                           |
  |_last_cpupid                      |
  +----------------------------------+
```

```
  folio
  page                          page_1
  +-----------------------------+--------------------------------+
  |flags                        |_flags_1(order)                 |
  |                             |_head_1(compound_head)          |
  |                             |_large_mapcount/_nr_pages_mapped|
  |mapping                      |_entire_mapcount/_pin_count     |
  |share                        |_mm_id_mapcount[2]              |
  |private                      |_mm_ids                         |
  |_mapcount                    |_mapcount_1                     |
  |_refcount                    |_refcount_1                     |
  |                             |_nr_pages                       |
  +-----------------------------+--------------------------------+
```

其中：

  * _mapcount:             单个page被映射的次数
  * _large_mapcount:       整个folio被映射的次数（只有large folio有）
  * _entire_mapcount:      整个folio被作为PMD/PUD整体映射的次数
  * _nr_pages_mapped:      整个folio被映射的页数，一个页面最多被计数一次。外加ENTIRELY_MAPPED

## folio->mapping

其中mapping表示映射到什么地方，分别有匿名映射和文件映射：

```
do_pte_missing()
    do_anonymous_page()                                   // 匿名页
        folio_add_new_anon_rmap()
            __folio_set_anon()
                WRITE_ONCE(folio->mapping, anon_vma)      <--- 设置为anon_vma
    do_fault()
        do_cow_fault()                                    // 文件页
            vma->vm_ops->fault() -> filemap_fault()
                file = vmf->vma->vm_file
                mapping = file->f_mapping
                __filemap_get_folio(mapping, index)
                    filemap_add_folio()
                        __filemap_add_folio()
                            folio->mapping = mapping      <--- 设置为mapping
```

# 增加和减少rmap的函数

在rmap.c文件中，有很多处理rmap的函数。让我在这里整理一下：

增加rmap

```
文件页：

    folio_add_file_rmap_ptes
    folio_add_file_rmap_pmd
    folio_add_file_rmap_pud
        __folio_add_file_rmap
            __folio_add_rmap

匿名页：
    folio_add_anon_rmap_ptes
    folio_add_anon_rmap_pmd
        __folio_add_anon_rmap
            __folio_add_rmap
    folio_add_new_anon_rmap
```

删除rmap

```
   folio_remove_rmap_ptes
   folio_remove_rmap_pmd
   folio_remove_rmap_pud
        __folio_remove_rmap
```

从上面的整理后可以看出，rmap最核心的两个函数是__folio_add_rmap和__folio_remove_rmap。然后还有个特例folio_add_new_anon_rmap，个人认为逻辑上是一样的，不过省去了一些检测。

* 另外值的注意的一点是hugetlb的rmap是在别的地方处理的。

## __folio_add_rmap

有必要展开看看这两个函数，距离上一次看明白大概过了两个月，现在又不明白了。

首先看看不是large folio的情况。这个比较简单，直接增加_mapcount计数就行了。并且因为__mapcount从-1开始，所以如果增加后为0,则会变更folio stat。

再看看large folio。这里又分两种情况： PTE map 和 非PTE map。

### PTE map

这里我们只看!CONFIG_NO_PAGE_MAPCOUNT的情况。

  * 增加所有page的_mapcount计数
  * 增加folio->_nr_pages_mapped计数
  * 增加folio->_large_mapcount计数
  * 调整folio stats

其中第一点比较好理解，第二点其实也还行，不过增加的数量是page数量而不是folio的数量(因为这里是作为单个page映射的)。比较绕的是第三、四点。

我们对照这代码来看看。

```
	atomic_t *mapped = &folio->_nr_pages_mapped;

		do {
			first += atomic_inc_and_test(&page->_mapcount);
		} while (page++, --nr_pages > 0);

		if (first &&
		    atomic_add_return_relaxed(first, mapped) < ENTIRELY_MAPPED)
			nr = first;

		folio_add_large_mapcount(folio, orig_nr_pages, vma);

	__folio_mod_stat(folio, nr, nr_pmdmapped);
```

  1. 首先遍历folio中的page，增加page->_mapcount。顺便看看其中多少是第一次被映射的 -- first。
  2. 如果有第一次被映射的，然后才会去考虑调整folio->_nr_pages_mapped。因为这个值表示的是folio中有多少页被映射过，每个页面只计数一次。
  3. 增加folio->_large_mapcount计数 nr_pages，这样对于large folio，可以直接读取这个值得到映射数，而不用遍历所有page。
  4. 第四点和非PTE map相关，也就是为了简化_nr_pages_mapped表达，其中做了一个边界值ENTIRELY_MAPPED。如果已经设置过这个值了，说明folio被整体映射过，那也就不需要调整统计值了。 当然也可以不这么做，每次都检查一下folio->_entire_mapcount。但是这么做会多计算。

### 非PTE map

我们同样只看!CONFIG_NO_PAGE_MAPCOUNT的情况。

  * 增加folio->_entire_mapcount
  * folio->_nr_pages_mapped计数 + ENTIRELY_MAPPED
  * 增加folio->_large_mapcount
  * 调整folio stats

其中第二点值得解释。因为在调整统计值的时候，我们调整的是有多少新增页被映射。虽然我们映射了一整个folio，但是为了这个目的，我们要知道之前已经映射了多少。而这个值要通过folio->_nr_pages_mapped取低位来得到。

另外第三点，folio->_large_mapcount在非PTE映射时，只计数1,而PTE映射计数nr_pages。语义上表示这个folio在页表中占多少page table entry.

## __folio_remove_rmap

这个过程就是__folio_add_rmap()反过来。值得一提的是partially_mapped的判定。

当CONFIG_NO_PAGE_MAPCOUNT时，这个判断是精准的。而不是的时候，这个判断是一个概率值。因为folio->_large_mapcount在PTE level，每个page都会映射。
所以如果多个page被重复映射，这个值即便是大于folio_large_nr_pages()也可能是partially_mapped。

# maybe mapped shared

后来内核中又发展出来一个尝试简化判断页面是否只被一个进程映射的方法(exclusive)。

这个系列有点多，重要的是这个改动。

```
commit 6af8cb80d3a9a6bbd521d8a7c949b4eafb7dba5d
Author: David Hildenbrand <david@redhat.com>
Date:   Mon Mar 3 17:30:05 2025 +0100

    mm/rmap: basic MM owner tracking for large folios (!hugetlb)
```

其中最关键的改动在函数folio_add_return_large_mapcount()/folio_sub_return_large_mapcount()。

而最后想要达到的目标是在folio_maybe_mapped_shared()，通过test_bit(FOLIO_MM_IDS_SHARED_BITNUM, &folio->_mm_ids)来判断有没有可能是被shared mapped。

当然这个只是**可能**，也就是如果这个bit设置，那么可能是共享映射。但是如果这个bit没有设置，那么肯定是独享映射。

[1]: /mm/10-page_struct.md
[2]: https://lwn.net/Articles/974223/

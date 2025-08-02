当分配内存给进程使用，被映射到进程页表中时，内核会在page中记录page被映射的次数。也是因为有大页的存在，这个计数的过程变得有点复杂。

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
  +-----------------------------+-------------------------------+
  |flags                        |_flags_1(order)                |
  |                             |_head_1(compound_head)         |
  |                             |large_mapcount/_nr_pages_mapped|
  |mapping                      |_entire_mapcount/_pin_count    |
  |share                        |_mm_id_mapcount[2]             |
  |private                      |_mm_ids                        |
  |_mapcount                    |_mapcount_1                    |
  |_refcount                    |_refcount_1                    |
  |                             |_nr_pages                      |
  +-----------------------------+-------------------------------+
```

## folio->mapping

其中mapping表示映射到什么地方，分别有匿名映射和文件映射：

```
do_pte_missing()
    do_anonymous_page()
        folio_add_new_anon_rmap()
            __folio_set_anon()
                WRITE_ONCE(folio->mapping, anon_vma)
    do_fault()
        do_cow_fault()
            vma->vm_ops->fault() -> filemap_fault()
                __filemap_get_folio()
                    filemap_add_folio()
                        __filemap_add_folio()
                            folio->mapping = mapping
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

[1]: /mm/10-page_struct.md

当分配内存给进程使用，被映射到进程页表中时，内核会在page中记录page被映射的次数。也是因为有大页的存在，这个计数的过程变得有点复杂。

# 都在哪里计数

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

[1]: /mm/10-page_struct.md

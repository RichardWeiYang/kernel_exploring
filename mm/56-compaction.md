除了内存回收，当系统内存吃紧时，系统还会做内存规整。通过将内存尽量迁移到高位整理出大块的可用内存。

其核心函数是compact_zone()。

```
页面分配失败 (__alloc_pages_slowpath)
   └>__alloc_pages_direct_compact()       后台后台线程 (kcompactd)
         └>compact_zone_order()                 └>kcompactd_do_work()
               │                                        │
               └────────┬───────────┘
                                 v
			 compact_zone()  <-- [核心控制大脑]
                                  │
      ┌-------------------------------------------------------┐
      │  while (compact_finished() == COMPACT_CONTINUE) {     │ <- [主外层循环]
      │                                                       │
      │   1. isolate_migratepages()                           │ <- [步骤 1: 锁定源数据]
      │        └->isolate_migratepages_block()               │
      │              └->把可移动页塞入 cc->migratepages 链表 │
      │                                                       │
      │   2. isolate_freepages()                              │ <- [步骤 2: 搜刮目标房子]
      │        └->isolate_freepages_block()                  │
      │              └->把空闲页塞入 cc->freepages 链表      │
      │                                                       │
      │   3. migrate_pages(&cc->migratepages, ...)            │ <- [步骤 3: 复制与解绑]
      │        └->真正迁移数据，更新页表                     │
      │                                                       │
      │   4. release_freepages(&cc->freepages)                │ <- [步骤 4: 清理收尾]
      │        └->没用完的空闲页退还给 Buddy System          │
      │  }                                                    │
      └-------------------------------------------------------┘
```

解释一点，步骤2 发生在步骤3 的过程中，migrate_pages()通过isolate_freepages()获得可用的内存空间用于迁移目标内存。

从内存地址空间的角度看

```
Zone 低地址 (low_pfn)                                               Zone 高地址 (high_pfn)
 ┌-----------------------------------------------------------------------┐
 │   Migration Scanner  |       未扫描区域        |     Free Scanner     |
 └-----------------------------------------------------------------------┘
 1. isolate_migratepages()                           2. isolate_freepages()
   (寻找被占用的“可移动页”)                       (寻找“空闲页”作为目标)
             │                                              │
             └-------------------+--------------------------┘
                                  |
                                  v
	                 3. migrate_pages()
                         (把数据 Copy 过去)
```

上面的流程中，我们可以看到alloc_contig_frozen_range_noprof()中使用了[isolate相关函数][1]中提到的一组api。

  * isolate_migratepages() / isolate_freepages()

而且这两个函数作用的物理地址范围是不一样的: 一个是在zone的前段，一个是在zone的后段,这一点和[cma][2]不一样。

[1]: /mm/15-pageblock_migratetype.md
[2]: /mm/55-cma.md

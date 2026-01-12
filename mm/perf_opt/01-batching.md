有时候批量处理会给性能带来很大提升。这里总结一下相关的优化。

# 批量清理PTE

```
commit 789753e17c4d6593932f07e40b740373123296a6
Author: David Hildenbrand <david@kernel.org>
Date:   Wed Feb 14 21:44:26 2024 +0100

    mm/memory: factor out zapping of present pte into zap_present_pte()
```

# 批量unmap lazy free large folio

```
commit 354dffd29575cdf13154e8fb787322354aa9efc4
Author: Barry Song <baohua@kernel.org>
Date:   Fri Feb 14 22:30:14 2025 +1300

    mm: support batched unmap for lazyfree large folios during reclamation


commit ddd05742b45b083975a0855ef6ebbf88cf1f532a
Author: Lance Yang <lance.yang@linux.dev>
Date:   Fri Jun 27 14:23:19 2025 +0800

    mm/rmap: fix potential out-of-bounds page table access during batched unmap
```

# 批量迁移

最开始那个

```
6f7d760e86fa 2023-02-16 migrate_pages: move THP/hugetlb migration support check to simplify code
7e12beb8ca2a 2023-02-16 migrate_pages: batch flushing TLB
ebe75e475106 2023-02-16 migrate_pages: share more code between _unmap and _move
80562ba0d837 2023-02-16 migrate_pages: move migrate_folio_unmap()
5dfab109d519 2023-02-16 migrate_pages: batch _unmap and _move
64c8902ed441 2023-02-16 migrate_pages: split unmap_and_move() to _unmap() and _move()
42012e0436d4 2023-02-16 migrate_pages: restrict number of pages to migrate in batch
e5bfff8b10e4 2023-02-16 migrate_pages: separate hugetlb folios migration
5b855937096a 2023-02-16 migrate_pages: organize stats with struct migrate_pages_stats
```

后来发现有dead lock

```
commit fb3592c41a4427601f9643b2a84e55bb99f5cd7c
Author: Ying Huang <huang.ying.caritas@gmail.com>
Date:   Fri Mar 3 11:01:53 2023 +0800

    migrate_pages: fix deadlock in batched migration

2ef7dbb26990 2023-03-07 migrate_pages: try migrate in batch asynchronously firstly
a21d2133215b 2023-03-07 migrate_pages: move split folios processing out of migrate_pages_batch()
fb3592c41a44 2023-03-07 migrate_pages: fix deadlock in batched migration
```

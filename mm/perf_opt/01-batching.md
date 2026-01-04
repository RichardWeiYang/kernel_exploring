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

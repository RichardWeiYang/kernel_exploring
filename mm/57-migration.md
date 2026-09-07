内存迁移对外的接口是migrate_pages()

```
migrate_pages(from, get_new_folio, put_new_folio, private, mode, reason, ret_succeeded)
    list_cut_before(&folios, from, &folio2->lru)                               // 先从from中切出一部分到folios
    migrate_pages_batch(&folios, get_new_folio, put_new_folio, private, MIGRATE_ASYNC,
                        reason, &ret_folios, &split_folios, &stat, NR_MAX_MIGRATE_PAGES_RETRY)
    ...or...
    migrate_pages_sync(&folios, get_new_folio, put_new_folio, private, mode,
                       reason, &ret_folios, &split_folios, &stat)
        migrate_pages_batch(...)
```

所以其中干活的关键是migrate_pages_batch()

```
migrate_pages_batch(from, get_new_folio, put_new_folio, private, mode, reason,
                    ret_folios, split_folios, stats, nr_pass)
    list_for_each_entry_safe(src, folio2, from, lru)                           // 遍历from中的每一个folio

    // 阶段1： unmap，将页表设置为migration entry, 移除映射
    migrate_folio_unmap(get_new_folio, put_new_folio, private, src, &dst,
                        mode, ret_folios)
        dst = get_new_folio(src, private)                            // dst.refcount = 1
        folio_trylock(src)                                           // 锁住原folio
        folio_trylock(dst)                                           // 锁住目标folio

        try_to_migrate(src, )                                        // 替换成migration entry, 解除映射

    // 如果成功移除映射
    list_move_tail(&folio->lru, &unmap_folios)
    list_add_tail(&dst->lru, &dst_folios)

    try_to_unmap_flush()

    migrate_folios_move(&unmap_folios, &dst_folios, put_new_folio, private,    // 复制内容，并map
                        mode, reason, ret_folios, stats, &retry, &thp_retry,
                        &nr_failed, &nr_retry_pages)

    // 阶段2： move，复制内容到新页面，并映射
        migrate_folio_move(put_new_folio, private, src, dst, mode, reason, ret_folios)

            move_to_new_folio(dst, src, mode)
                migrate_folio(mapping, dst, src, mode) -> __migrate_folio()
                    // 引用计数只能是多一个，也就是migrate_pages()前拿到的计数
                    folio_ref_count(src) != folio_expected_ref_count(src) + 1
                    folio_mc_copy(dst, src)
                    __folio_migrate_mapping(mapping, dst, src, expected_count)
                    folio_migrate_flags(dst, src)

            remove_migration_ptes(src, dst, 0)                                 // 替换为新的页面
                remove_migrate_pte()
                    folio_get(dst)                                   // dst.refcount = 2

            folio_unlock(dst)
            folio_put(dst)                                           // dst.refcount = 1

            list_del(&src->lru)
            folio_unlock(src)
            folio_put(src)                                           // 匿名页src.refcount = 0
```


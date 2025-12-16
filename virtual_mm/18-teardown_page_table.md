页表在必要的时候还需要释放：比如进程销毁或者munmap。

看过了页表构造的过程（虽然糙了点），那也该看看页表释放的过程。 在页表释放的时候，其实有四件事情要做：

  * 清除PMD/PTE等页表项
  * tlb flush，告诉其他cpu取消(shootdown)对应的映射
  * 释放进程页
  * 释放页表页

这一点在内核代码中include/asm-generic/tlb.h开头的注释中也得到了印证。并且值的注意的是，一定要在tlb flush后再释放进程页，避免进程依旧访问导致错误。

下面这个函数是我找到的接口之一，让我们从这个接口入手分析一下页表释放的过程。

```
vms_clear_ptes(vms, mas_detach, mm_wr_locked)
    struct mmu_gather tlb;
    tlb_gather_mmu(&tlb, vms->vma->vm_mm)
        __tlb_reset_range(tlb)
        inc_tlb_flush_pending(mm)                                         // mm->tlb_flush_pending++
    // iterate mas_detach to call unmap_single_vma()
    unmap_vmas(&tlb, mas_detach, vms->vma, vms->start, vms->end, vms->vma_count)
        unmap_single_vma(tlb, vma, start, end, &details)
            unmap_page_range(tlb, vma, start, end, detail)
                tlb_start_vma(tlb, vma)
                zap_p4d_range(tlb, vma, pgd, addr, next, )
                    // now comes to the PTE table
                    zap_pte_range(tlb, vma, pmd, addr, next, details)
                tlb_end_vma(tlb, vma)
                    tlb_flush_mmu_tlbonly(tlb)

    free_pgtables(&tlb, mas_detach, vms->vma, vms->unmap_start, vms->unmap_end, )
        tlb_free_vmas(tlb)
        free_pgd_range(tlb, addr, vma->vm_end, floor, )
            tlb_change_page_size(tlb, PAGE_SIZE)
            free_p4d_range() ...  free_pmd_range()
                free_pte_range(tlb, pmd, addr)
                    pmd_clear(pmd);
                    pte_free_tlb(tlb, token, addr)
                        tlb_flush_pmd_range(tlb, address, PAGE_SIZE);
                        tlb->freed_tables = 1;
                        __pte_free_tlb(tlb, ptep, address); ->  tlb_remove_table()
    tlb_finish_mmu(&tlb)
        dec_tlb_flush_pending(mm)
```


```
zap_pte_range(tlb, vma, pmd, addr, end, details)
    tlb_change_page_size(tlb, PAGE_SIZE)

    pte = pte_offset_map_lock(mm, pmd, addr, &ptl)    // lock page table

    flush_tlb_batched_pending(mm)                     // arch want batched unmap tlb flush
        flush_tlb_mm(mm) -> flush_tlb_mm_range()      // if pending
    lazy_mmu_mode_enable() -> arch_enter_lazy_mmu_mode()

        do_zap_pte_range(tlb, vma, pte, addr, end, details, )
            zap_present_ptes(tlb, vma, pte, ptent, max_nr, addr, end, )
                nr = folio_pte_batch(folio, pte, ptent, max_nr)
                zap_present_folio_ptes(tlb, vma, folio, page, pte, ptent, nr, )

                    clear_full_ptes(mm, addr, pte, nr, );                             // 清理PTE
                    tlb_remove_tlb_entries(tlb, pte, nr, addr)                        // ?
                    folio_remove_rmap_ptes(folio, page, nr, vma)                      // 去除反向映射
                    __tlb_remove_folio_pages(tlb, page, nr, delay_remap)              // 释放进程页(也就是pte映射的页)

            or

            zap_nonpresent_ptes(tlb, )

    lazy_mmu_mode_disable()
    if (force_flush) {
        tlb_flush_mmu_tlbonly(tlb);
        tlb_flush_rmaps(tlb, vma);
    }

    pte_unmap_unlock(, &ptl)                          // unlock page table
```


# mmu_gather

在整个页表释放的过程中，有一个贯穿始终的结构体mmu_gather，也就是函数中第一个参数tlb。

我们需要好好看看这个结构。

```
    struct mmu_gather
    +------------------------------+
    |mm                            |
    |   (struct mm_struct*)        |
    |start/end                     |
    |   (unsigned long)            |
    |page_size                     |   // granularity of zap?
    |   (unsigned int)             |
    |                              |
    |batch                         |   // __get_free_page()分配
    |   (struct mmu_table_batch*)  |
    |   +--------------------------+
    |   |nr                        |   // 攒够了一起tlb_table_flush()
    |   |   (unsigned int)         |
    |   |tables[]                  |
    |   |   (void *)               |
    |   +--------------------------+
    |                              |
    |batch_count                   |   // 已经分配了多少mmu_gather_batch
    |   (unsigned int)             |
    |*active                       |   // active = &local
    |local                         |
    |   (struct mmu_gather_batch)  |
    |   +--------------------------+
    |   |next                      |
    |   |(struct mmu_gather_batch*)|
    |   |                          |
    |   |nr/max                    |
    |   |   (unsigned int)         |
    |   |encoded_pages[]           |
    |   |   (void *)               |
    |   +--------------------------+
    |__pages[]                     |   // 给上面的local用作encoded_pages[]的
    |   (struct page *)            |
    |                              |
    |// indicators                 |
    |                              |
    |fullmm                        |
    |need_flush_all                |
    |                              |
    |freed_tables                  |    // removed page directories
    |                              |
    |cleared_ptes                  |    // the level we cleared
    |cleared_pmds                  |
    |cleared_puds                  |
    |cleared_p4ds                  |
    |                              |
    +------------------------------+

```

# API分析

在内核代码中include/asm-generic/tlb.h开头的注释中，分类注释了相关的API。我们接下来展开学习一下。

## 释放进程页

进程页的释放分成两个阶段：

  * 将进程页cache到一个地方 -- __tlb_remove_folio_pages_size()等
  * 释放之前cache的进程页  -- __tlb_batch_free_encoded_pages()

其中进程页cache在mmu_gather::active中。

下面展示一下这部分相关的数据结构。

```
    struct mmu_gather
    +------------------------------+
    |                              |
    |batch_count                   |   // 已经分配了多少mmu_gather_batch
    |   (unsigned int)             |
    |*active                       |   // active = &local
    |local                         |
    |   (struct mmu_gather_batch)  |
    |   +--------------------------+
    |   |next                      |
    |   |(struct mmu_gather_batch*)|
    |   |                          |
    |   |nr/max                    |
    |   |   (unsigned int)         |
    |   |encoded_pages[]           |
    |   |   (void *)               |
    |   +--------------------------+
    |__pages[]                     |   // 给上面的local用作encoded_pages[]的
    |   (struct page *)            |
    |                              |
    +------------------------------+
```

函数__tlb_remove_folio_pages_size()将page存放到mmu_gather::active中，然后在__tlb_batch_free_encoded_pages()中将所有存放的页释放。

另外有意思的是，__tlb_batch_free_encoded_pages()一定会通过tlb_flush_mmu()调用到。锁是释放进程页的链路比较清晰。

但是将进程页加入到batch的链路就略微有点多了：

```
__tlb_remove_folio_pages_size() 是最核心              // 直接调用者只有__tlb_remove_folio_pages() 和 __tlb_remove_page_size()

zap_present_folio_ptes()                              // 只有这个可能delay_rmap，并且由caller执行tlb_flush_mmu()
    __tlb_remove_folio_pages()                        // only used in zap_present_folio_ptes()
        __tlb_remove_folio_pages_size(, PAGE_SIZE)

tlb_remove_page_size(, page, page_size)               // tlb_remove_page() / zap_huge_pmd() / zap_huge_pud()
    __tlb_remove_page_size(, page_size)
        __tlb_remove_folio_pages_size(, 1, false, page_size)
    tlb_flush_mmu()                                   // 如果batch满了，就释放

tlb_remove_page(tlb, page)
    tlb_remove_page_size(, PAGE_SIZE)
zap_huge_pmd()
    tlb_remove_page_size(, &folio->page, HPAGE_PMD_SIZE)
zap_huge_pud()
    tlb_remove_page_size(, &folio->page, HPAGE_PUD_SIZE)
__unmap_hugepage_range()
    tlb_remove_page_size(, &folio->page, folio_size(folio))
```

整理完后其实也比较清晰，我们用一个图来展示一下：

```
                  __tlb_remove_folio_pages_size()
                          /            \
         may delay_remap /              \ no delay_remap
                        /                \
    __tlb_remove_folio_pages()         __tlb_remove_page_size() 
              |                                  |
              |                                  |
              |                                  |
    zap_present_folio_ptes()           tlb_remove_page_size()
                                            /           \
                                          /              \
                                        /                 \
                        tlb_remove_page()                 zap_huge_pmd()
                               |                          zap_huge_pud()
                               |                          __unmap_hugepage_range()
                               |
                        tlb_remove_table()
```


## 释放页表页

tlb_flush_mmu_free()


```
tlb_remove_table(tlb, table)
    tlb_table_flush(tlb)
        tlb_table_invalidate(tlb)
            tlb_flush_mmu_tlbonly(tlb)
                tlb_flush(tlb)                                 // x86 implementation
                    flush_tlb_mm_range(tlb->mm, start, end, )  // 可刷可不刷
                __tlb_reset_range(tlb)
        tlb_remove_table_free(tlb->batch)
```


























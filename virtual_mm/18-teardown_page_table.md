页表在必要的时候还需要释放：比如进程销毁或者munmap。

看过了页表构造的过程（虽然糙了点），那也该看看页表释放的过程。 在页表释放的时候，其实有四件事情要做：

  * 清除PMD/PTE等页表项
  * tlb flush，告诉其他cpu取消(shootdown)对应的映射
  * 释放进程页
  * 释放页表页

这一点在内核代码中include/asm-generic/tlb.h开头的注释中也得到了印证。并且值的注意的是，一定要在tlb flush后再释放进程页，避免进程依旧访问导致错误。


# mmu_gather

在整个页表释放的过程中，有一个贯穿始终的结构体mmu_gather，也就是函数中第一个参数tlb。所以我们首先来认识一下这个数据结构。

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
    |batch                         |   // 专门给页表页释放做缓存
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

# 释放进程页

进程页的释放分成两个阶段：

  * 将进程页cache到缓冲区 -- __tlb_remove_folio_pages_size()等
  * 释放之前cache的进程页  -- __tlb_batch_free_encoded_pages()

其中进程页cache在mmu_gather::active中。

下面展示一下这部分相关的数据结构。

```
    struct mmu_gather
    +------------------------------+
    |                              |
    |batch_count                   |   // 已经分配了多少mmu_gather_batch
    |   (unsigned int)             |
    |*active                    ---|-+ // active = &local
    |local                         | |    // 最多MAX_GATHER_BATCH_COUNT个
    |   (struct mmu_gather_batch)  | +--->+---------+---------+---------+
    |   +--------------------------+      |         |         |         |
    |   |next                      |      +---------+---------+---------+
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

## 缓存进程页

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
                        v                v
    __tlb_remove_folio_pages()         __tlb_remove_page_size()
              |                                  |
              |                                  |
              v                                  v
    zap_present_folio_ptes()           tlb_remove_page_size()
                                            /           \
                                          /              \
                                        v                 v
                        tlb_remove_page()                 zap_huge_pmd()
                               |                          zap_huge_pud()
                               |                          __unmap_hugepage_range()
                               v
                        tlb_remove_table()
```

## 释放进程页

释放的流程相对简单一些，最核心的是__tlb_batch_free_encoded_pages()。我们也画一个调用关系。

```
                  __tlb_batch_free_encoded_pages()
                              |
                              |
                              v
                      tlb_batch_pages_flush()
                              |
                              |
                              v
                        tlb_flush_mmu_free()
                              |
                              |
                              v
                          tlb_flush_mmu()
                          /     |         \
                      /         |              \
                  v             v                  v
    tlb_remove_page_size()  tlb_finish_mmu()  tlb_change_page_size()
    zap_pte_range()
```

从上面的调用路径可以看到有几个时间点去释放进程页

  * 清理页表过程中： tlb_remove_page_size() / zap_pte_range()
  * 清理过程结束时： tlb_finish_mmu()


# 释放页表页

释放页表页的核心函数是tlb_remove_table()。

## 缓存页表页

释放页表页的行为根据CONFIG_MMU_GATHER_TABLE_FREE的配置有两种情况：

  * 使用普通的缓存释放机制：tlb_remove_page(), !CONFIG_MMU_GATHER_TABLE_FREE
  * 使用特殊的缓存释放机制：mmu_table_batch, CONFIG_MMU_GATHER_TABLE_FREE

使用普通缓存释放机制的不用多说了，和进程页的释放逻辑一致。

这里我们来看看使用mmu_table_batch情况下，是怎么处理的。

```
tlb_remove_table(tlb, table)
    batch = &tlb->batch;
    if (!batch)                       // 如果没有缓冲区，先分配
    if (batch->nr == MAX_TABLE_BATCH) // 如果满了flush
        tlb_table_flush(tlb)
```

所以原理上差不多，先存，后释放。

接下来我们看看都在哪里会调用tlb_remove_table().

```
                      tlb_remove_table()
                              |
                              |
                              v
                      tlb_remove_ptdesc()
                              |
                              |
                              v
           ___pte_free_tlb()/__pte_free_tlb()/pte_free_tlb()  --> free_pte_range()
                                                                       |
                                                                       v
           ___pmd_free_tlb()/__pmd_free_tlb()/pmd_free_tlb()  --> free_pmd_range()
                                                                       |
                                                                       v
           ___pud_free_tlb()/__pud_free_tlb()/pud_free_tlb()  --> free_pud_range()
                                                                       |
                                                                       v
           ___p4d_free_tlb()/__p4d_free_tlb()/p4d_free_tlb()  --> free_p4d_range()
                                                                       |
                                                                       v
                                                                  free_pgtables()
```

也就是在释放页表过程中，页表页会放到缓存中。如果缓存满了，就会调用tlb_table_flush()去最后释放。

## 释放页表页

这里主要研究一下tlb_table_flush()的逻辑。

```
tlb_table_flush(tlb)
    tlb_table_invalidate(tlb)
        tlb_needs_table_invalidate(tlb)             // 默认返回true，不同arch可以改
            /*
             * Invalidate page-table caches used by hardware walkers. Then
             * we still need to RCU-sched wait while freeing the pages
             * because software walkers can still be in-flight.
             */
            tlb_flush_mmu_tlbonly(tlb)
    tlb_remove_table_free(batch)                    // 释放页表页，也可能通过rcu
        __tlb_remove_table_free(batch)
            pagetable_dtor_free()
                __free_pages()
```

值得注意的是，先刷新tlb，再释放页表页。

此外，tlb_table_flush()不仅在tlb_remove_table()中被调用，还有在tlb_flush_mmu_free()中调用。是的，也就是在tlb_flush_mmu()是也会被调用，而且是在释放进程页之前。


```
            tlb_flush_mmu()
                  |
                  |
                  v
          tlb_flush_mmu_free()              tlb_remove_table()
                 \                                 /
                      \                        /
                          v                v
                            tlb_table_flush()
                          /
                      /
                 v
          tlb_batch_pages_flush()
```

从上面的调用关系可以看出，释放进程页和页表页时，都会要释放页表页。而且在释放进程页之前，先要释放页表页；并且在此之前先要做tlb flush。

这就和我们开头说的顺序对上了。

# 刷新tlb

刷新tlb的核心是tlb_flush(), 但是它被封装在函数tlb_flush_mmu_tlbonly()。

除了上面看到的在释放进程页和页表页之前需要刷新tlb外，还有几个地方需要。按照我的立即，总的来说有两类：

  * zap的时候
  * free的时候

# 代码分析

对API有了解了后，我们展开实际的代码看看是怎么个过程。比如unmap一段空间时，将vma划分好后，会将需要释放的vma添加到mas_detach，然后调用vms_clear_ptes()去释放对应的进程页和页表页。

```
vms_clear_ptes(vms, mas_detach, mm_wr_locked)                    // 此时对应的vma已经从进程中删除
    struct mmu_gather tlb;
    tlb_gather_mmu(&tlb, vms->vma->vm_mm)
        __tlb_reset_range(tlb)
        inc_tlb_flush_pending(mm)                                // mm->tlb_flush_pending++

    // 清理 mas_detach 中对应的空间
    unmap_vmas(&tlb, mas_detach, vms->vma, vms->start, vms->end, vms->vma_count)
     unmap_single_vma(tlb, vma, start, end, &details)
      unmap_page_range(tlb, vma, start, end, detail)             // 一直清理完整个vma才返回
          tlb_start_vma(tlb, vma)

          zap_p4d_range(tlb, vma, pgd, addr, next, )
           // now comes to the PTE table
           zap_pte_range(tlb, vma, pmd, addr, next, details)
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
                               tlb_remove_tlb_entries(tlb, pte, nr, addr)                        // 记录范围到tlb->start/end
                               folio_remove_rmap_ptes(folio, page, nr, vma)                      // 去除反向映射，!tlb->delayed_rmap
                               __tlb_remove_folio_pages(tlb, page, nr, delay_remap)              // 将进程页保存到batch缓存

                       or

                       zap_nonpresent_ptes(tlb, )

               lazy_mmu_mode_disable()
               if (force_flush) {
                   tlb_flush_mmu_tlbonly(tlb);                   // 刷tlb
                   tlb_flush_rmaps(tlb, vma);                    // 去除反向映射，tlb->delayed_rmap
               }

               pte_unmap_unlock(, &ptl)                          // unlock page table


               if (force_flush)
                   tlb_flush_mmu(tlb)                            // 实际上只要释放进程页就好了，tlb已经flush过了


          tlb_end_vma(tlb, vma)
              tlb_flush_mmu_tlbonly(tlb)                         // 清理完整个vma，刷tlb

    // 释放 mas_detach 中vma的对应的页表页
    free_pgtables(&tlb, mas_detach, vms->vma, vms->unmap_start, vms->unmap_end, )
        tlb_free_vmas(tlb)
        free_pgd_range(tlb, addr, vma->vm_end, floor, )          // 可能合并多个vma一起free
            tlb_change_page_size(tlb, PAGE_SIZE)
            free_p4d_range(tlb, pgd, addr, end, )
                free_pud_range(tlb, p4d, addr, next, floor, )

                    free_pte_range(tlb, pmd, addr)
                        pmd_clear(pmd)                           // 清除pmd entry
                        pte_free_tlb(tlb, token, addr)           // 释放对应的页表页

                pgd_clear(pgd, addr)
                p4d_free_tlb(tlb, p4d, start)                    // 释放p4d对应的页表页

    tlb_finish_mmu(&tlb)
        tlb_flush_mmu(tlb)
            tlb_flush_mmu_tlbonly(tlb)                           // 刷tlb
            tlb_flush_mmu_free(tlb)
                tlb_table_flush()                                // 释放页表页，必要时刷新tlb
                tlb_batch_pages_flush()                          // 释放进程页
        tlb_batch_list_free(tlb)                                 // 释放mmu_gather_batch
        dec_tlb_flush_pending(mm)
```

## free_pte_range必定释放一个PMD(2M)空间

看到下面这段代码的时候我一开始一直都有一个困惑：

```
free_pmd_range
    pmd = pmd_offset(pud, addr)
    if (pmd_none_or_clear_bad(pmd))
        continue;
    free_pte_range(tlb, pmd, addr)
        pmd_clear(pmd)                  <--- 不是来清理PTE的么，怎么就把pmd给干掉了？
```

这个问题要回到zap_pmd_range里。

```
zap_pmd_range
    pmd = pmd_offse(pud, addr);
    if (pmd_is_huge(*pmd))
        zap_huge_pmd(tlb, vma, pmd, addr);
    if (pmd_none(*pmd))
        continue;
    addr = zap_pte_range(tlb, vma, pmd, addr, next, details)
```

在这里我们看到：

  * 如果是huge pmd，那就整个都回收了
  * 如果不是，实际上只释放了pte对应的进程页和清理了pte，但是对应的页表页没有清理

所以free_pmd_range中只会遇到两种情况：pmd_none_or_clear_bad()和pmd_is_huge()。

## CONFIG_MMU_GATHER_RCU_TABLE_FREE 和 CONFIG_PT_RECLAIM

在释放页表页过程中，有两个相关的配置选项控制了释放的流程:

  * CONFIG_MMU_GATHER_RCU_TABLE_FREE
  * CONFIG_PT_RECLAIM

这两个配置的作用都是使对应的函数有一个rcu的版本。他们分别控制的函数是：

  * tlb_remove_table_free()
  * tlb_remove_table_one()

这两个函数只有两处被调用。

第一处是tlb_finish_mmu()链路上。

```
tlb_finish_mmu(tlb)
    tlb_flush_mmu(tlb)
        tlb_flush_mmu_free(tlb)
            tlb_table_flush(tlb)
                tlb_remove_table_free(tlb->batch)          <--- here
            tlb_batch_pages_flush(tlb)
```

第二处是在free_pte_range()链路上。

```
free_pte_range(tlb, addr)
    pte_free_tlb/__pte_free_tlb/___pte_free_tlb
        tlb_remove_ptdesc(tlb, pt)
            tlb_remove_table(tlb, pt)

                // 如果tlb->batch分配不出，只能单个释放页表页时
                tlb_table_invalidate(tlb)
                tlb_remove_table_one(table)                // <--- here

                // 如果tlb->batch分配出，然后缓存满了时
                tlb_table_flush(tlb)
                    tlb_table_invalidate(tlb)
                    tlb_remove_table_free(tlb->batch)      // <--- here
```

总结一下，这两个函数都是在释放进程页的时候会被调用。区别是如果内存紧张，没有办法分配mmu_table_batch，那么就直接调用 tlb_remove_table_one()释放单个页，而不是先queue到batch中然后一次性释放多个。

### MMU_GATHER_RCU_TABLE_FREE

我们来学习一下为什么要有MMU_GATHER_RCU_TABLE_FREE，以及这个的作用是什么。

```
commit 267239116987d64850ad2037d8e0f3071dc3b5ce
Author: Peter Zijlstra <a.p.zijlstra@chello.nl>
Date:   Tue May 24 17:12:00 2011 -0700

    mm, powerpc: move the RCU page-table freeing into generic code

    In case other architectures require RCU freed page-tables to implement
    gup_fast() and software filled hashes and similar things, provide the
    means to do so by moving the logic into generic code.
```

这个功能原来是在powerpc上的，Peter搬到了通用代码。

从commit log上看，这个功能和gup_fast()有关。这个commit中的一段注释，帮助我理解代码的作用。现在这部分注释放到了mmu_gather.c，CONFIG_MMU_GATHER_RCU_TABLE_FREE定义的代码首部。

```
  * This is needed by some architectures to implement software pagetable walkers.
  *
  * gup_fast() and other software pagetable walkers do a lockless page-table
  * walk and therefore needs some synchronization with the freeing of the page
  * directories. The chosen means to accomplish that is by disabling IRQs over
  * the walk.
  *
  * Architectures that use IPIs to flush TLBs will then automagically DTRT,
  * since we unlink the page, flush TLBs, free the page. Since the disabling of
  * IRQs delays the completion of the TLB flush we can never observe an already
  * freed page.
  *
  * Architectures that do not have this (PPC) need to delay the freeing by some
  * other means, this is that means.
  *
  * What we do is batch the freed directory pages (tables) and RCU free them.
  * We use the sched RCU variant, as that guarantees that IRQ/preempt disabling
  * holds off grace periods.
  *
  * However, in order to batch these pages we need to allocate storage, this
  * allocation is deep inside the MM code and can thus easily fail on memory
  * pressure. To guarantee progress we fall back to single table freeing, see
  * the implementation of tlb_remove_table_one().
```

从上面的注释中，我们得到几点信息：

  * gup_fast()是软件扫描页表的动作，需要和硬件页表内容同步
  * 第二段：gup_fast()采用的是无锁机制进行页表访问
  * 第三段：有些硬件通过关中断来实现和硬件同步，并解释了为什么能做到
  * 第四段：但是有些硬件不能这样，因为没有用IPI来flush tlb，所以诞生了RCU_TABLE_FREE，现在名字叫MMU_GATHER_RCU_TABLE_FREE。
  * 第五段：对于这种硬件，通过rcu释放来避免冲突。因为关中段期间是一个grace period。如果grace period结束，页表释放时，表明了之前的pgtable walker已经结束了流程，所以不会访问到已经释放的页表页。
  * 第六段：如果分配不出mmu_table_batch，那就只free当前一个。有意思的是这时后的版本只有通过发送ipi同步gup_fast()。

这么看，这个选项需要根据不同硬件系统来配置。对于没有IPI tlb flush的，必须选。而其他的硬件系统，可选可不选。

PS: 我还学到一个东西，tlb flush IPI是会等目标cpu完成IPI任务的。

### PT_RECLAIM

配置文件mm/Kconfig中明确写了PT_RECLAIM 依赖 MMU_GATHER_RCU_TABLE_FREE。 PT_RECLAIM的出现要从这个series说起。

```
synchronously scan and reclaim empty user PTE pages

4817f70c25b6 2025-01-13 x86: select ARCH_SUPPORTS_PT_RECLAIM if X86_64
718b13861d22 2025-01-13 x86: mm: free page table pages by RCU instead of semi RCU
6375e95f381e 2025-01-13 mm: pgtable: reclaim empty PTE page in madvise(MADV_DONTNEED)
2686d514c345 2025-01-13 mm: make zap_pte_range() handle full within-PMD range
4059971c79fc 2025-01-13 mm: do_zap_pte_range: return any_skipped information to the caller
735fad44b5a8 2025-01-13 mm: zap_install_uffd_wp_if_needed: return whether uffd-wp pte has been re-installed
45fec1e59514 2025-01-13 mm: skip over all consecutive none ptes in do_zap_pte_range()
117cdb05e32d 2025-01-13 mm: introduce do_zap_pte_range()
fabc0e8dac5b 2025-01-13 mm: introduce zap_nonpresent_ptes()
dd95d2782a25 2025-01-13 mm: userfaultfd: recheck dst_pmd entry in move_pages_pte()
6c18ec9af86c 2025-01-13 mm: khugepaged: recheck pmd state in retract_page_tables()
```

不过最相关的要属这个：

```
commit 6375e95f381e3dc85065b6f74263a61522736203
Author: Qi Zheng <zhengqi.arch@bytedance.com>
Date:   Wed Dec 4 19:09:49 2024 +0800

    mm: pgtable: reclaim empty PTE page in madvise(MADV_DONTNEED)
```

原因： madvise(MADV_DONTNEED or MADV_FREE) 释放了进程页但是没有释放页表页
解法： 据说是探测到整个PTE都是空，就清pmd。

但是这么做之后，引入了一个新问题：

之前MMU_GATHER_RCU_TABLE_FREE，解决的是gup_fast()和硬件页表之间的同步。gup_fast()采用的是关中断的方式，现在我们在zap_page_range_single()中引入了新的软件遍历页表的情况，且没有关中断。这个时候怎么保证软件和硬件之间没有冲突呢？

那就只这个commit的作用了。

```
718b13861d22 2025-01-13 x86: mm: free page table pages by RCU instead of semi RCU
```

好消息是pte_offset_map()用了rcu_read_lock()来保护，所以干脆用PT_RECLAIM的时候，同一用rcu同步就行了。

OK, 我觉得我理解了，准备除去打怪了。

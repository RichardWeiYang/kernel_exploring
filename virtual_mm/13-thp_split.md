# 拆分页表

现在的透明大页还支持拆分，这样可以在必要的时候退回到四级页表。

这个工作由函数__split_huge_pmd_locked完成。

```
__split_huge_pmd_locked
    pgtable = pgtable_trans_huge_withdraw(mm, pmd);
    pmd_populate(mm, &_pmd, pgtable);
    for (i = 0; i < HPAGE_PMD_NR; i++) {
        set_pte_at(mm, addr, pte, entry);
    }
    pmd_populate(mm, pmd, pgtable);
```

其实说白了很简单，就是把预留了页表拿出来，每一项都填上大页对应的地址。最后把相应的pmd改好。是不是有点酷。

当然我这个流程中只显示了匿名页的拆分过程，对于文件页还需要继续学习。

# 拆分大页

拆分页表最终的目的也是为了拆分大页，这样在系统内存不够时可以回收大页中没有使用的部分。

下图是一个非常简化版本的大致流程。

```
    split_huge_page_to_list
        unmap_page
            try_to_unmap_one
                ...
                __split_huge_pmd_locked
        __split_huge_page
            remap_page
```

可以看到，拆分大页的过程中，也有可能会拆分页表。此时我们称之为PMD-mapped THP。如果这个大页的页表已经被拆分过了，就不需要也不能再次拆分，我们就叫它PTE-mapped THP。

所以对于匿名大页，整个拆分的过程就分成两种情况

  * PMD-mapped THP
  * PTE-mapped THP

我们分别来看看这两种情况下拆分过程中细微的差别。

对于PMD-mapped THP, __split_huge_pmd_locked函数会执行到。该函数会将PTE entry改写成migration entry。然后在函数remap_page中再将migration entry 恢复到页表中。怎么样，是不是很有意思。

对于PTE-mapped THP, __split_huge_pmd_locked函数不会被执行，因为页表早已拆分。此时try_to_unmap_one函数就担负起将PTE entry设置成migration entry的重任。接下来就和之前一样，由remap_page将migration entry恢复到页表中。

# 手动拆分

为了测试split的功能，内核还提供一个调试接口文件/sys/kernel/debug/split_huge_pages。按照格式写入该文件，则会触发对应页面的split。

PS: 也可以参考tools/testing/selftests/mm/split_huge_page_test.c中的使用方法。

## 使用方法

这个文件在mm/huge_memory.c中创建的，只有写的接口，没有读的。对应的方法是split_huge_pages_write()。

写这个文件分为两种格式： 文件和进程的。这次我们只看进程的。

在split_huge_page_test.c中有定义这个格式：

```
#define PID_FMT "%d,0x%lx,0x%lx,%d"
```

也就是 pid, start_vaddr, end_vaddr, new_order

可以看到，因为现在有mTHP了，所以拆分的时候可以指定拆分到的页面大小。当new_order省略时，默认位0。

# deferred_split_shrinker

系统运行时一般情况下我们不会去手动拆分大页，而是通过shrinker扫描将浪费的大页拆分 -- deferred_split_shrinker。

shrinker的机制不在本章范围，简单来说函数do_shrink_slab()会在必要的时候调用count_objects/scan_objects。而deferred_split_shrinker的这两个成员是：

  * deferred_split_count
  * deferred_split_scan

前者就是查看对应ds_queue->split_queuue_len，后者则需要执行关键的操作split_folio()。

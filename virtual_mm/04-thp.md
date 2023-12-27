虚拟内存管理中有个常用，非常有意思，但是知道名字但是不知道是怎么运作的机制--透明大页(Transparent Huge Page)。

之前我对这个东西也不是很理解，而且还经常和另一个概念混淆--hugetlb。这两个都是利用了页表中“大页”表项，减少tlb的失误进而提高内存访问速度。而两者不同的是，相对hugetlb， THP更加灵活，当然也更加复杂。

# 相同之处

THP和hugetlb相同之处，或者说和其他虚拟内存的相同之处在于在缺页中断中的处理流程几乎是一样的。在__handle_mm_fault函数中会逐级查询，并按照不同情况进行处理。比如如果pmd为空，且支持THP，那么就会调用create_huge_pmd()来创建。或者是在COW的情况下，就会调用wp_huge_pmd()进行处理。

```
__handle_mm_fault
    create_huge_pmd()
    do_huge_pmd_numa_page
    wp_huge_pmd
```

# 不同之处

相同之处平平无奇，关键还是在于不同之处。所谓透明，其实就是比hugetlb多了一些灵活性。在必要的时候可以拆分页表或者拆分大页本身。

就当前的代码阅读理解来看，有这么几个不同之处。

* 预留页表
* 拆分页表
* 拆分大页

## 预留页表

在创建大页表项的时候，与其他人不同，还预留了一张页表。分别对应两个函数。

* pgtable_trans_huge_deposit()
* pgtable_trans_huge_withdraw()

我们先来看第一个。这个函数的动作是预留了一个页表。那什么时候用呢？就是在第二个函数发生的时候。那为什么要这么做呢？这个就牵扯到下一个不同点--拆分页表。

我们知道对于大页，页表一共有三级。而正常的4k页的页表是四级。所以当我们要拆分大页页表时，就需要补上这么一级。为了保证拆分时，不会因为内存不够导致不能展开到四级页表，所以在分配时就多预留了一个页表。

不得不说这是一个有意思的设计。

## 拆分页表

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

## 拆分大页

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

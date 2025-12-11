当THP功能打开后，内核会启动一个专门的线程khugepaged，将原本不是大页的页表合并成大页。


# 加入扫描队列

khugepaged作为一个线程会不停扫描自己的队列，看是否有需要合并的内存。我们先来看看这个队列的结构。

```
    khugepaged_scan
    +---------------------------+
    |address                    |  当前扫描到的虚拟地址
    |   (unsigned long)         |
    |mm_slot                    |  当前正在扫描的进程信息
    |   (khugepaged_mm_slot *)  |
    |   +-----------------------+
    |   |slot                   |
    |   |   (struct mm_slot)    |
    |   |   +-------------------+
    |   |   |mm                 |  指向进程的mm_struct
    |   |   |                   |
    |   |   |mm_node            |  添加到khugepaged_scan.mm_head的节点
    |   |   | (struct list_head)|
    |   |   |hash               |  添加到mm_slots_hash表的节点
    |   |   | (hlist_node)      |
    |   +-----------------------+
    |                           |
    |mm_head                    |  一个用于存放khugepaged_mm_slot.slot的链表
    |   (struct list_head)      |
    |                           |
    +---------------------------+

    mm_slots_hash
    +---------------------------+
    |                           |  一个用于存放khugepaged_mm_slot.slot的hash表
    +---------------------------+

```

将需要扫描的进程加入队列的函数是khugepaged_enter_vma()。每次都会分配一个khugepaged_mm_slot结构，然后将这个结构添加到khugepaged_scan.mm_head和mm_slots_hash中。

# 扫描&合并

为了有一个整体的感觉，我们把中间重要的流程抽取出来。

首先是扫描的过程：

```
collapse_scan_mm_slot()                     扫描某一个进程
    collapse_single_pmd()                   扫描进程中某一段PMD范围
        collapse_scan_pmd()                 扫描进程中一段匿名PMD范围
	    // 扫描PMD中的pte，符合条件后继续
            collapse_huge_page()            尝试合并
```

其次是合并的过程：

```
collapse_huge_page()
    alloc_charge_folio()                    分配好替换内存
    __collapse_huge_page_swapin()           如果有swap entry，先把内存调进来
    __collapse_huge_page_isolate()
    __collapse_huge_page_copy()             将内容复制到分配出的替换内存中
```

## 进程级扫描

khugepaged的扫描可以分为两个级别

  * 进程级
  * vma级

首先我们来看进程级扫描，collapse_scan_mm_slot()。这个级别主要是在遍历khugepaged.mm_head。

初始的时候，khugepaged_scan.mm_slot是空的，所以它会从khugepaged.mm_head上取下一个，写到khugepaged.mm_slot，开始做vma级扫描。

当这个进程的所有vma都扫描完后，则从khugepaged.mm_head上取出下一个，循环往复。

## vma级扫描

vma级扫描也是在函数collapse_scan_mm_slot()里，不过是在for_each_vma()这个循环中按照每2M的大小逐个扫描。


## PMD级扫描

最后就是按照PMD对齐进行扫描collapse_single_pmd()，其中又分了匿名页和文件页。

这次我们只看匿名页collapse_scan_pmd()。

整个扫描过程就是对PMD这块区间的每个pte进行检查，对于不符合要求的就跳过。

## 合并

到这里，我们已经找到了合适的目标，接下来就是将这部分pmd用一个thp来代替，这个工作交给了collapse_huge_page()。

合并的过程又可以分为

  * swapin: __collapse_huge_page_swapin()
  * isolate: __collapse_huge_page_isolate()
  * copy: __collapse_huge_page_copy()

先把swapout的页面拿进来，然后隔离，最后把内容拷贝到新分配的大页。如果成功了，将原先的页面释放，否则恢复原先页面。

# 测试

针对大页合并，内核中提供了一个用户态的测试程序。

tools/testing/selftests/mm/khugepaged.c

这算是[selftests][1]的一个例子。

[1]: /mm/tests/01_functional_test.md
